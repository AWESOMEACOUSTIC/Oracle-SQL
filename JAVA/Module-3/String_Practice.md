# Java Strings, StringBuilder and StringBuffer: Assessment Practice Set

**Focus:** `String` · `StringBuilder` · `StringBuffer`
**Style:** company scenario → requirements → given driver code → samples → answer → reasoning

## How to use this set

* Same format as the collections set: the **driver code is given** and you write only the classes or methods named in the requirements. Each driver is shown so you can run it locally.
* **Attempt each question before scrolling to its answer.** Samples show the normal path; hidden assessment tests will hit the edge cases listed in each *Reasoning* section (blank input, boundaries, case rules, `-1` from `indexOf`, empty results).
* Every reference solution was compiled and run against the sample inputs, and the outputs shown are the real program output.
* **Java version:** solutions avoid Java 11+ methods (`repeat`, `isBlank`, `strip`, `StringBuilder.compareTo`) so they run on older assessment platforms. The Reasoning sections mention the newer shortcuts.
* Several samples end with an empty line or contain leading/trailing spaces on purpose. The drivers print results inside `[ ]` so you can see them.
* Java's natural `String` ordering compares character codes, so **uppercase sorts before lowercase** (`"Z" < "a"`).

## Contents

| # | Question | Focus | Level |
|---|---|---|---|
| 1 | [NexaCorp Username and Email Masking](#q1-nexacorp-username-and-email-masking) | String (trim, split, substring, indexOf) | Easy |
| 2 | [WordPlay Palindrome and Anagram Checker](#q2-wordplay-palindrome-and-anagram-checker) | String (charAt, toCharArray, sorting characters) | Easy-Medium |
| 3 | [CityLibrary Catalog Code Comparison](#q3-citylibrary-catalog-code-comparison) | String (equals, equalsIgnoreCase, compareTo, charAt) | Easy-Medium |
| 4 | [SecureLogin Password and Email Validation](#q4-securelogin-password-and-email-validation) | String (contains, startsWith/endsWith, indexOf/lastIndexOf, Character methods) | Medium |
| 5 | [PressNet Headline Formatter](#q5-pressnet-headline-formatter) | String (split, join, substring, toUpperCase/toLowerCase) | Medium |
| 6 | [TextScan Character Statistics](#q6-textscan-character-statistics) | String (Character methods, indexOf, toCharArray) | Medium |
| 7 | [OpsWatch Log and File Parser](#q7-opswatch-log-and-file-parser) | String (indexOf with fromIndex, lastIndexOf, substring) | Medium |
| 8 | [RoadGuard Regex Validators](#q8-roadguard-regex-validators) | String (matches, replaceAll, groups and back-references) | Medium-Hard |
| 9 | [BillMate Invoice Formatting](#q9-billmate-invoice-formatting) | String (String.format, replace/replaceFirst, parsing) | Medium |
| 10 | [PackRight Serial Code Formatter](#q10-packright-serial-code-formatter) | StringBuilder (insert, append, reverse, setCharAt) | Easy-Medium |
| 11 | [DocuClean Text Cleaner](#q11-docuclean-text-cleaner) | StringBuilder (deleteCharAt, delete, indexOf, charAt, setCharAt) | Medium |
| 12 | [AuditTrail Report Builder](#q12-audittrail-report-builder) | StringBuilder (setLength, length, equals trap, in-place editing) | Medium-Hard |
| 13 | [TicketDesk Shared Log (StringBuffer)](#q13-ticketdesk-shared-log-stringbuffer) | StringBuffer (thread safety) | Medium-Hard |
| 14 | [ArchiveX Run-Length Compression](#q14-archivex-run-length-compression) | StringBuilder (append(char), append(int), parsing digits) | Hard |
| 15 | [CipherDesk Caesar Cipher and Rotation](#q15-cipherdesk-caesar-cipher-and-rotation) | StringBuilder + String (char arithmetic, contains) | Medium |
| 16 | [SearchLab Longest Word and Unique Window](#q16-searchlab-longest-word-and-unique-window) | String + StringBuilder (split with regex, sliding window) | Hard |

---

# Part A: String

## Q1. NexaCorp Username and Email Masking

**Focus:** String (trim, split, substring, indexOf)  ·  **Level:** Easy

NexaCorp, an IT company in Bengaluru, is setting up accounts for new joiners. The system must generate a username from the employee's full name and mask the email address before displaying it on the screen. Write a Java program based on the following requirements. The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static String generateUsername(String fullName)
```

* Ignore leading, trailing and repeated spaces in the name.
* If the name has two or more words, the username is the **first letter of the first word** followed by the **last word**, all in lowercase.
* If the name has a single word, the username is that word in lowercase.
* If the name is empty or contains only spaces, return `"INVALID"`.

**Requirement 2**

```java
public static String maskEmail(String email)
```

Keep the first **two** characters of the part before `@` (or all of them if it is shorter than two) and replace every remaining character of that part with `*`. The `@` and the domain remain unchanged.
If the email has no `@`, or nothing before the `@`, return `"INVALID"`.

**Input Format**
Line 1: the full name (may contain spaces). Line 2: the email address.

**Output Format**
Line 1: `Username: <username>`. Line 2: `Email: <masked email>`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String fullName = sc.nextLine();
        String email = sc.nextLine();
        System.out.println("Username: " + UserMainCode.generateUsername(fullName));
        System.out.println("Email: " + UserMainCode.maskEmail(email));
    }
}
```

**Sample Input 1**

```text
  Rahul   Kumar Sharma 
arjun.k@nexacorp.com
```

**Sample Output 1**

```text
Username: rsharma
Email: ar*****@nexacorp.com
```

**Sample Input 2**

```text
PRIYA
ab@x.in
```

**Sample Output 2**

```text
Username: priya
Email: ab@x.in
```

**Sample Input 3**

```text
     
no-at-sign
```

**Sample Output 3**

```text
Username: INVALID
Email: INVALID
```

**Sample Input 4**

```text
Meena Iyer
@nexacorp.com
```

**Sample Output 4**

```text
Username: miyer
Email: INVALID
```

### Answer

```java
// UserMainCode.java
public class UserMainCode {
    public static String generateUsername(String fullName) {
        String trimmed = fullName.trim();
        if (trimmed.isEmpty()) {
            return "INVALID";
        }
        String[] words = trimmed.split("\\s+");
        if (words.length == 1) {
            return words[0].toLowerCase();
        }
        return (words[0].charAt(0) + words[words.length - 1]).toLowerCase();
    }

    public static String maskEmail(String email) {
        int at = email.indexOf('@');
        if (at <= 0) {
            return "INVALID";
        }
        String local = email.substring(0, at);
        String domain = email.substring(at);

        int keep = Math.min(2, local.length());
        StringBuilder sb = new StringBuilder(local.substring(0, keep));
        for (int i = keep; i < local.length(); i++) {
            sb.append('*');
        }
        sb.append(domain);
        return sb.toString();
    }
}
```

### Reasoning

* **`trim()` removes only the ends**, so the repeated spaces inside `"Rahul   Kumar"` remain. That is why the split uses the regex `"\\s+"` (one or more whitespace characters) rather than `" "`. Splitting on a single space would produce empty strings between the words.
* **Check blank input before splitting.** `"     ".trim()` is `""`, and `"".split("\\s+")` returns an array holding one empty string, so `words[0].charAt(0)` would throw `StringIndexOutOfBoundsException`. The `isEmpty()` guard prevents that. (On Java 11+, `isBlank()` does the trim-and-check in one call.)
* **`char + String` is concatenation**: `words[0].charAt(0) + words[last]` works because one operand is a `String`. If both operands were `char`, `+` would add their numeric codes instead.
* **Lowercase once, at the end**, so the initial and the surname are handled in one call.
* **`indexOf('@')` returns `-1` when absent**, and `0` when `@` is the first character. `at <= 0` covers both invalid cases in a single check (Samples 3 and 4).
* **`substring(begin, end)`** includes `begin` and excludes `end`. `email.substring(at)` (one argument) takes everything from `@` onwards, so the domain keeps its `@`.
* **`Math.min(2, local.length())`** protects short local parts (Sample 2, `ab`) from `substring(0, 2)` going out of range on a one-character name.
* **Java 11+ shortcut**: `"*".repeat(local.length() - keep)` replaces the loop. The loop version is shown because older assessment platforms may run Java 8.


---

## Q2. WordPlay Palindrome and Anagram Checker

**Focus:** String (charAt, toCharArray, sorting characters)  ·  **Level:** Easy-Medium

WordPlay Studios, a game developer based in Mumbai, is building a word puzzle game. The game needs two checks on player input. Write a Java program based on the following requirements. The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static boolean isPalindrome(String s)
```

Return `true` if the string reads the same forwards and backwards, considering **only letters and digits** and **ignoring case**. Spaces and punctuation are skipped. A string with no letters or digits counts as a palindrome.

**Requirement 2**

```java
public static boolean areAnagrams(String a, String b)
```

Return `true` if the two strings are made of exactly the same letters with the same counts, **ignoring case and ignoring spaces**.

**Input Format**
Three lines: the text for the palindrome check, then the two strings for the anagram check.

**Output Format**
`Palindrome: Yes` or `Palindrome: No`, then `Anagrams: Yes` or `Anagrams: No`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String text = sc.nextLine();
        String a = sc.nextLine();
        String b = sc.nextLine();
        System.out.println("Palindrome: " + (UserMainCode.isPalindrome(text) ? "Yes" : "No"));
        System.out.println("Anagrams: " + (UserMainCode.areAnagrams(a, b) ? "Yes" : "No"));
    }
}
```

**Sample Input 1**

```text
A man, a plan, a canal: Panama
Listen
Silent
```

**Sample Output 1**

```text
Palindrome: Yes
Anagrams: Yes
```

**Sample Input 2**

```text
race a car
Dormitory
Dirty room
```

**Sample Output 2**

```text
Palindrome: No
Anagrams: Yes
```

**Sample Input 3**

```text
!!
aab
ab
```

**Sample Output 3**

```text
Palindrome: Yes
Anagrams: No
```

**Sample Input 4**

```text
No 'x' in Nixon
abc
ABD
```

**Sample Output 4**

```text
Palindrome: Yes
Anagrams: No
```

### Answer

```java
// UserMainCode.java
import java.util.Arrays;

public class UserMainCode {
    public static boolean isPalindrome(String s) {
        int left = 0;
        int right = s.length() - 1;

        while (left < right) {
            while (left < right && !Character.isLetterOrDigit(s.charAt(left))) {
                left++;
            }
            while (left < right && !Character.isLetterOrDigit(s.charAt(right))) {
                right--;
            }
            if (Character.toLowerCase(s.charAt(left)) != Character.toLowerCase(s.charAt(right))) {
                return false;
            }
            left++;
            right--;
        }
        return true;
    }

    public static boolean areAnagrams(String a, String b) {
        char[] x = a.replaceAll("\\s", "").toLowerCase().toCharArray();
        char[] y = b.replaceAll("\\s", "").toLowerCase().toCharArray();
        Arrays.sort(x);
        Arrays.sort(y);
        return Arrays.equals(x, y);
    }
}
```

### Reasoning

* **Two-pointer palindrome check** avoids building a cleaned copy: one index moves in from the left, one from the right, and each skips characters that are not letters or digits. It uses `O(1)` extra space. The easier alternative is to clean first (`s.replaceAll("[^A-Za-z0-9]", "").toLowerCase()`) and compare with `new StringBuilder(cleaned).reverse().toString()`. Know both.
* **The inner `while` loops need `left < right`** in their conditions. Without it, a string like `"!!"` would let the pointers cross and `charAt` would run out of range. The empty-after-cleaning case (Sample 3) then falls straight through to `true`.
* **`Character.isLetterOrDigit`** is the safe test for "keep this character". Digits count, so `"0P"` is **not** a palindrome even though a cleaned letters-only version would look like one.
* **Compare with `Character.toLowerCase(...)`** on both sides. Comparing `s.charAt(left) != s.charAt(right)` directly would treat `'A'` and `'a'` as different.
* **Anagram = same multiset of characters.** Sorting both character arrays and comparing them (`Arrays.equals`) is the standard approach. An alternative is counting letters in an `int[26]`.
* **`replaceAll("\\s", "")`** removes all whitespace. `replace(" ", "")` would also work here but only for plain spaces, not tabs.
* **Do not use `==` or `.equals` on arrays**: `x == y` compares references and `x.equals(y)` does too. `Arrays.equals(x, y)` compares the contents.
* **Different lengths short-circuit** naturally: arrays of different sizes are never `Arrays.equals` (Sample 4 differs by a letter, Sample 3 differs by length).


---

## Q3. CityLibrary Catalog Code Comparison

**Focus:** String (equals, equalsIgnoreCase, compareTo, charAt)  ·  **Level:** Easy-Medium

CityLibrary, a public library in Chennai, tags every book with a catalog code such as `BK-102`. The cataloguing team needs a few helpers to compare and order these codes. Write a Java program based on the following requirements. The `UserMainCode` class must contain three static methods.

**Requirement 1**

```java
public static String compareCodes(String a, String b)
```

Return exactly one of these strings:

* `"IDENTICAL"` if the two codes are exactly equal (case-sensitive).
* `"SAME_IGNORING_CASE"` if they differ only in letter case.
* `"FIRST_COMES_FIRST"` if the codes are different and `a` comes before `b` in natural (lexicographic, case-sensitive) order.
* `"SECOND_COMES_FIRST"` otherwise.

**Requirement 2**

```java
public static int commonPrefixLength(String a, String b)
```

Return the length of the longest common prefix of the two codes (case-sensitive). Do not use any built-in method that computes the prefix directly.

**Requirement 3**

```java
public static String[] sortCodes(String[] codes)
```

Return a **new** array with the codes sorted in ascending order, **ignoring case**. Codes that are equal ignoring case must keep their original relative order. The input array must not be modified.

**Input Format**
Two codes `a` and `b`, then `n`, then `n` codes.

**Output Format**
The result of Requirement 1, then `Common prefix length: <k>`, then the sorted codes on one line separated by a single space.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String a = sc.next();
        String b = sc.next();
        int n = sc.nextInt();
        String[] codes = new String[n];
        for (int i = 0; i < n; i++) {
            codes[i] = sc.next();
        }

        System.out.println(UserMainCode.compareCodes(a, b));
        System.out.println("Common prefix length: " + UserMainCode.commonPrefixLength(a, b));
        System.out.println(String.join(" ", UserMainCode.sortCodes(codes)));
    }
}
```

**Sample Input 1**

```text
BK-102 bk-102 4
b2 A1 a1 B1
```

**Sample Output 1**

```text
SAME_IGNORING_CASE
Common prefix length: 0
A1 a1 B1 b2
```

**Sample Input 2**

```text
BK-102 BK-205 3
zeta Alpha beta
```

**Sample Output 2**

```text
FIRST_COMES_FIRST
Common prefix length: 3
Alpha beta zeta
```

**Sample Input 3**

```text
app apple 1
X
```

**Sample Output 3**

```text
FIRST_COMES_FIRST
Common prefix length: 3
X
```

**Sample Input 4**

```text
X X 0
```

**Sample Output 4**

```text
IDENTICAL
Common prefix length: 1
```

### Answer

```java
// UserMainCode.java
import java.util.Arrays;

public class UserMainCode {
    public static String compareCodes(String a, String b) {
        if (a.equals(b)) {
            return "IDENTICAL";
        }
        if (a.equalsIgnoreCase(b)) {
            return "SAME_IGNORING_CASE";
        }
        return a.compareTo(b) < 0 ? "FIRST_COMES_FIRST" : "SECOND_COMES_FIRST";
    }

    public static int commonPrefixLength(String a, String b) {
        int limit = Math.min(a.length(), b.length());
        int i = 0;
        while (i < limit && a.charAt(i) == b.charAt(i)) {
            i++;
        }
        return i;
    }

    public static String[] sortCodes(String[] codes) {
        String[] copy = Arrays.copyOf(codes, codes.length);
        Arrays.sort(copy, String.CASE_INSENSITIVE_ORDER);
        return copy;
    }
}
```

### Reasoning

* **Order of the checks matters.** `equals` first, then `equalsIgnoreCase`, then `compareTo`. If you test `equalsIgnoreCase` first, exactly-equal codes would be reported as `SAME_IGNORING_CASE`.
* **`==` is not `equals`.** `==` compares object references. Two strings with identical content can be different objects (`new String("BK-102") == "BK-102"` is `false`), while `equals` compares the characters. String literals are pooled, which is why `==` sometimes "works" in small tests and then fails on input read from a `Scanner`.
* **`compareTo` returns a sign, not a fixed value**: negative, zero or positive. Always test `< 0`, `> 0` or `== 0`, never `== -1`. It compares character codes at the first differing position, or the **length difference** if one string is a prefix of the other. `"app".compareTo("apple")` is `-2`, so `app` comes first (Sample 3).
* **Uppercase sorts before lowercase** in natural order because `'A'` (65) is less than `'a'` (97). That is why the case-sensitive comparison of `BK-102` and `bk-102` is not the same as the ignore-case one.
* **`compareToIgnoreCase`** exists for a single comparison, and `String.CASE_INSENSITIVE_ORDER` is the ready-made `Comparator` for sorting. Both compare case-folded characters.
* **Stable sort**: `Arrays.sort` on object arrays is stable, so `A1` (index 1) stays before `a1` (index 2) in Sample 1, and `B1` stays before `b2` because `1 < 2`. Sorting primitives (`char[]`, `int[]`) has no stability concern since equal values are indistinguishable.
* **Prefix loop**: the bound `i < limit` is checked **before** the `charAt` calls, which prevents an `IndexOutOfBoundsException` when one string is shorter. `"app"` and `"apple"` give `3` because the loop stops at the end of the shorter string.
* **Copy before sorting**: `Arrays.sort` sorts in place, so sorting `codes` directly would violate the "input must not be modified" rule. `Arrays.copyOf` is a quick way to copy.
* **Empty array** (Sample 4): `String.join(" ", new String[0])` returns an empty string, so the driver prints a blank line without error.


---

## Q4. SecureLogin Password and Email Validation

**Focus:** String (contains, startsWith/endsWith, indexOf/lastIndexOf, Character methods)  ·  **Level:** Medium

SecureLogin, an identity-management company in Hyderabad, needs to validate sign-up details without using regular expressions. Write a Java program based on the following requirements. The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static String checkPasswordStrength(String pwd)
```

* If the password is **shorter than 8 characters**, contains a **space**, or contains the word `password` in **any letter case**, return `"WEAK"`.
* Otherwise count how many of these four kinds of characters it contains: an uppercase letter, a lowercase letter, a digit, and a special character from `!@#$%^&*()_+-=`.
* Return `"STRONG"` for all four kinds, `"MEDIUM"` for two or three kinds, and `"WEAK"` for fewer than two.

**Requirement 2**

```java
public static boolean isValidEmail(String email)
```

Return `true` only if **all** of these hold:

* It contains no spaces.
* It contains exactly one `@`, and it is neither the first nor the last character.
* The part after the `@` (the domain) contains at least one `.`, does not start or end with `.`, and does not contain two consecutive dots (`..`).

**Input Format**
Line 1: the password. Line 2: the email address.

**Output Format**
`Password strength: <result>` then `Email valid: Yes` or `Email valid: No`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String pwd = sc.nextLine();
        String email = sc.nextLine();
        System.out.println("Password strength: " + UserMainCode.checkPasswordStrength(pwd));
        System.out.println("Email valid: " + (UserMainCode.isValidEmail(email) ? "Yes" : "No"));
    }
}
```

**Sample Input 1**

```text
Passw0rd!
user@mail.com
```

**Sample Output 1**

```text
Password strength: STRONG
Email valid: Yes
```

**Sample Input 2**

```text
MyPASSWORD99!
user@@mail.com
```

**Sample Output 2**

```text
Password strength: WEAK
Email valid: No
```

**Sample Input 3**

```text
abcd1234
user@mail
```

**Sample Output 3**

```text
Password strength: MEDIUM
Email valid: No
```

**Sample Input 4**

```text
Ab1!
user@.com
```

**Sample Output 4**

```text
Password strength: WEAK
Email valid: No
```

**Sample Input 5**

```text
Hello World1!
us er@mail.com
```

**Sample Output 5**

```text
Password strength: WEAK
Email valid: No
```

**Sample Input 6**

```text
12345678
@mail.com
```

**Sample Output 6**

```text
Password strength: WEAK
Email valid: No
```

### Answer

```java
// UserMainCode.java
public class UserMainCode {
    public static String checkPasswordStrength(String pwd) {
        if (pwd.length() < 8 || pwd.contains(" ") || pwd.toLowerCase().contains("password")) {
            return "WEAK";
        }

        String specials = "!@#$%^&*()_+-=";
        boolean upper = false, lower = false, digit = false, special = false;
        for (char c : pwd.toCharArray()) {
            if (Character.isUpperCase(c)) {
                upper = true;
            } else if (Character.isLowerCase(c)) {
                lower = true;
            } else if (Character.isDigit(c)) {
                digit = true;
            } else if (specials.indexOf(c) != -1) {
                special = true;
            }
        }

        int kinds = (upper ? 1 : 0) + (lower ? 1 : 0) + (digit ? 1 : 0) + (special ? 1 : 0);
        if (kinds == 4) {
            return "STRONG";
        }
        return kinds >= 2 ? "MEDIUM" : "WEAK";
    }

    public static boolean isValidEmail(String email) {
        if (email.contains(" ")) {
            return false;
        }
        int at = email.indexOf('@');
        if (at <= 0 || at != email.lastIndexOf('@') || at == email.length() - 1) {
            return false;
        }
        String domain = email.substring(at + 1);
        return domain.contains(".")
                && !domain.startsWith(".")
                && !domain.endsWith(".")
                && !domain.contains("..");
    }
}
```

### Reasoning

* **Reject early, then analyse.** The three "always weak" rules are checked first with `length()`, `contains(" ")` and `contains("password")`, so the character loop only runs for candidates that could still qualify.
* **Case-insensitive `contains`** has no built-in form. The idiom is to lowercase the text first: `pwd.toLowerCase().contains("password")`. In Sample 2, `MyPASSWORD99!` is caught even though it is otherwise strong. Note that `Passw0rd!` (Sample 1) is **not** caught because the letter `o` is a zero.
* **`Character` methods** (`isUpperCase`, `isLowerCase`, `isDigit`, `isLetterOrDigit`) are cleaner than range checks such as `c >= 'A' && c <= 'Z'` and also work for non-ASCII letters. The `else if` chain matters: a character can only be one kind.
* **`indexOf(c) != -1`** is the quick "is this character in this set" test. `specials.indexOf(c)` avoids writing a long `||` chain.
* **Counting kinds with the ternary** `(flag ? 1 : 0)` keeps the scoring branch-free. Sample 3 (`abcd1234`) has lowercase and digits, so `kinds = 2` gives `MEDIUM`. Sample 4 fails the length rule despite having four kinds.
* **`indexOf` vs `lastIndexOf` for "exactly one `@`"**: if the first and last positions are the same index, the character occurs once. This avoids counting in a loop. `at <= 0` also covers "not found" (`-1`) and "at the start" (`0`) in one test, and `at == length - 1` covers "at the end".
* **Short-circuit `&&`** makes the domain check safe and readable, and `domain.contains("..")` handles consecutive dots. Sample 3 fails because the domain `mail` has no dot, and Sample 4 because `.com` starts with a dot.
* **Order of guards**: check `contains(" ")` before slicing so the later logic never has to think about spaces.


---

## Q5. PressNet Headline Formatter

**Focus:** String (split, join, substring, toUpperCase/toLowerCase)  ·  **Level:** Medium

PressNet, a news agency in Delhi, receives headlines typed with irregular spacing and capitalisation. The editors want two clean-up tools. Write a Java program based on the following requirements. The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static String capitalizeWords(String sentence)
```

Return the sentence with the first letter of each word in **uppercase** and the remaining letters in **lowercase**. Leading and trailing spaces are removed, and any run of spaces between words becomes a **single space**.

**Requirement 2**

```java
public static String reverseWords(String sentence)
```

Return the words of the sentence in **reverse order**, separated by a single space. Leading, trailing and repeated spaces are ignored. The letters of each word keep their original case.

For an empty or blank sentence, both methods return an empty string.

**Input Format**
A single line containing the headline.

**Output Format**
`Capitalized: [<result>]` then `Reversed: [<result>]`. The square brackets make leading and trailing spaces visible.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String sentence = sc.nextLine();
        System.out.println("Capitalized: [" + UserMainCode.capitalizeWords(sentence) + "]");
        System.out.println("Reversed: [" + UserMainCode.reverseWords(sentence) + "]");
    }
}
```

**Sample Input 1**

```text
  hELLO   wORLD from   java 
```

**Sample Output 1**

```text
Capitalized: [Hello World From Java]
Reversed: [java from wORLD hELLO]
```

**Sample Input 2**

```text
single
```

**Sample Output 2**

```text
Capitalized: [Single]
Reversed: [single]
```

**Sample Input 3**

```text
     
```

**Sample Output 3**

```text
Capitalized: []
Reversed: []
```

**Sample Input 4**

```text
MARKETS rally   AS oil  slips
```

**Sample Output 4**

```text
Capitalized: [Markets Rally As Oil Slips]
Reversed: [slips oil AS rally MARKETS]
```

### Answer

```java
// UserMainCode.java
import java.util.*;

public class UserMainCode {
    public static String capitalizeWords(String sentence) {
        String trimmed = sentence.trim();
        if (trimmed.isEmpty()) {
            return "";
        }
        String[] words = trimmed.split("\\s+");
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < words.length; i++) {
            if (i > 0) {
                sb.append(' ');
            }
            sb.append(Character.toUpperCase(words[i].charAt(0)))
              .append(words[i].substring(1).toLowerCase());
        }
        return sb.toString();
    }

    public static String reverseWords(String sentence) {
        String trimmed = sentence.trim();
        if (trimmed.isEmpty()) {
            return "";
        }
        String[] words = trimmed.split("\\s+");
        Collections.reverse(Arrays.asList(words));
        return String.join(" ", words);
    }
}
```

### Reasoning

* **`trim()` then `split("\\s+")`** is the standard pair for "words separated by any amount of whitespace". Skipping `trim()` is the classic bug: a sentence with a leading space splits into an array whose **first element is an empty string**, and the result gets an extra space (or `charAt(0)` fails on `""`).
* **Guard the blank sentence** (Sample 3): `"".split("\\s+")` gives `[""]`, a one-element array with an empty string, so `charAt(0)` would throw. Returning early avoids it.
* **Capitalising one word**: `Character.toUpperCase(w.charAt(0)) + w.substring(1).toLowerCase()`. The `substring(1)` call is safe even for a one-letter word, because `"a".substring(1)` is `""` (start index equal to the length is allowed). `"a".substring(2)` would throw.
* **Strings are immutable**: `toUpperCase()`, `substring()` and `trim()` all return **new** strings. Calling `word.toUpperCase();` and ignoring the result changes nothing.
* **Why `StringBuilder` for the loop**: repeated `result = result + word` builds a new `String` each time (`O(n^2)` overall for long text). One builder appends in place.
* **`Collections.reverse(Arrays.asList(words))`** reverses the array in place, because `Arrays.asList` is a live view over the array. `String.join(" ", words)` then inserts exactly one space between the words. This works for `String[]` but not for `int[]` (primitive arrays are not converted to a list of elements).
* **Reversed keeps the original case**: Sample 1 gives `java from wORLD hELLO`, while the capitalised form is `Hello World From Java`. Do not build the reversed result from the already-capitalised words.


---

## Q6. TextScan Character Statistics

**Focus:** String (Character methods, indexOf, toCharArray)  ·  **Level:** Medium

TextScan Labs, a language-analytics company in Pune, profiles text messages. Write a Java program based on the following requirements. The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static int[] classifyCharacters(String input)
```

Return an `int` array of size 5 holding, in this order:

1. number of **vowels** (`a e i o u`, either case)
2. number of **consonants** (all other letters)
3. number of **digits**
4. number of **whitespace** characters
5. number of **other** characters (punctuation and symbols)

**Requirement 2**

```java
public static String removeDuplicateChars(String input)
```

Return the string with repeated characters removed, keeping only the **first** occurrence of each character and preserving the original order. The comparison is **case-sensitive** (`'a'` and `'A'` are different).

**Input Format**
A single line of text (it may be empty or contain spaces).

**Output Format**
Five lines `Vowels: x`, `Consonants: x`, `Digits: x`, `Spaces: x`, `Others: x`, then `Unique: [<result>]`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String input = sc.nextLine();

        int[] stats = UserMainCode.classifyCharacters(input);
        System.out.println("Vowels: " + stats[0]);
        System.out.println("Consonants: " + stats[1]);
        System.out.println("Digits: " + stats[2]);
        System.out.println("Spaces: " + stats[3]);
        System.out.println("Others: " + stats[4]);
        System.out.println("Unique: [" + UserMainCode.removeDuplicateChars(input) + "]");
    }
}
```

**Sample Input 1**

```text
Hello World 2026!
```

**Sample Output 1**

```text
Vowels: 3
Consonants: 7
Digits: 4
Spaces: 2
Others: 1
Unique: [Helo Wrd206!]
```

**Sample Input 2**

```text
abracadabra
```

**Sample Output 2**

```text
Vowels: 5
Consonants: 6
Digits: 0
Spaces: 0
Others: 0
Unique: [abrcd]
```

**Sample Input 3**

```text
R2-D2 & C-3PO
```

**Sample Output 3**

```text
Vowels: 1
Consonants: 4
Digits: 3
Spaces: 2
Others: 3
Unique: [R2-D &C3PO]
```

**Sample Input 4**

```text
Aa Aa
```

**Sample Output 4**

```text
Vowels: 4
Consonants: 0
Digits: 0
Spaces: 1
Others: 0
Unique: [Aa ]
```

### Answer

```java
// UserMainCode.java
public class UserMainCode {
    public static int[] classifyCharacters(String input) {
        int[] result = new int[5];
        for (char c : input.toCharArray()) {
            if (Character.isLetter(c)) {
                if ("aeiouAEIOU".indexOf(c) != -1) {
                    result[0]++;
                } else {
                    result[1]++;
                }
            } else if (Character.isDigit(c)) {
                result[2]++;
            } else if (Character.isWhitespace(c)) {
                result[3]++;
            } else {
                result[4]++;
            }
        }
        return result;
    }

    public static String removeDuplicateChars(String input) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < input.length(); i++) {
            char c = input.charAt(i);
            if (sb.indexOf(String.valueOf(c)) == -1) {
                sb.append(c);
            }
        }
        return sb.toString();
    }
}
```

### Reasoning

* **Classify with an `if / else if` chain**, ordered so that each character lands in exactly one bucket: letter (then vowel or consonant), digit, whitespace, everything else. Testing "consonant" as "not a vowel" would wrongly count digits and punctuation, which is the most common mistake here.
* **`"aeiouAEIOU".indexOf(c) != -1`** is a compact membership test. The alternative is `Character.toLowerCase(c)` followed by a comparison against `'a'`, `'e'` and so on.
* **`Character.isLetter` vs `isAlphabetic` vs a range check**: `isLetter` handles all letters in Unicode. Range checks like `c >= 'a' && c <= 'z'` silently ignore letters such as `é`.
* **`Character.isWhitespace`** covers spaces, tabs and newlines, so the count is not limited to `' '`.
* **Duplicate removal with a `StringBuilder`**: the builder itself acts as the "seen" record, because `indexOf` searches what has been kept so far. Sample 2 `abracadabra` becomes `abrcd`. The alternative is a `HashSet<Character>` for `O(1)` membership on long inputs. For short strings the `indexOf` approach is simpler.
* **`StringBuilder.indexOf` takes a `String`**, not a `char`, so `String.valueOf(c)` is required (`indexOf(c)` with a `char` does not compile on `StringBuilder`, in contrast with `String.indexOf(char)`).
* **Case sensitivity** (Sample 4): `"Aa Aa"` keeps `A`, `a` and the space, giving `Aa `. The trailing space is why the output is bracketed.
* **`"R2-D2 & C-3PO"`** (Sample 3) mixes digits, hyphens and an ampersand. Only spaces count as whitespace, and `-` and `&` count as "others".


---

## Q7. OpsWatch Log and File Parser

**Focus:** String (indexOf with fromIndex, lastIndexOf, substring)  ·  **Level:** Medium

OpsWatch, a monitoring company in Noida, processes server log lines and uploaded file names. Write a Java program based on the following requirements. The `UserMainCode` class must contain three static methods.

**Requirement 1**

```java
public static String extractModule(String logLine)
```

A log line looks like `2026-09-29 14:35:07 ERROR [PaymentService] Timeout occurred`. Return the text between the **first `[`** and the **first `]` that comes after it**. Return `"UNKNOWN"` if there is no such pair, or if the brackets are empty.

**Requirement 2**

```java
public static String getFileExtension(String fileName)
```

Return the extension in **lowercase**, without the dot: the text after the **last** `.`. Return an empty string if the name has no dot, if the only dot is the first character (a hidden file such as `.gitignore`), or if the name ends with a dot.

**Requirement 3**

```java
public static int countOccurrences(String text, String pattern)
```

Return how many times `pattern` occurs in `text`. Occurrences must **not overlap** and matching is case-sensitive. An empty pattern returns `0`. Do not use `split` or `replace`.

**Input Format**
Four lines: the log line, the file name, the text, and the pattern.

**Output Format**
`Module: <result>`, `Extension: [<result>]`, `Count: <result>`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String logLine = sc.nextLine();
        String fileName = sc.nextLine();
        String text = sc.nextLine();
        String pattern = sc.nextLine();

        System.out.println("Module: " + UserMainCode.extractModule(logLine));
        System.out.println("Extension: [" + UserMainCode.getFileExtension(fileName) + "]");
        System.out.println("Count: " + UserMainCode.countOccurrences(text, pattern));
    }
}
```

**Sample Input 1**

```text
2026-09-29 14:35:07 ERROR [PaymentService] Timeout occurred
Report.Final.PDF
the cat sat on the mat with the hat
the
```

**Sample Output 1**

```text
Module: PaymentService
Extension: [pdf]
Count: 3
```

**Sample Input 2**

```text
no module here
.gitignore
aaaa
aa
```

**Sample Output 2**

```text
Module: UNKNOWN
Extension: []
Count: 2
```

**Sample Input 3**

```text
bad ] then [X] end
file.
abc
z
```

**Sample Output 3**

```text
Module: X
Extension: []
Count: 0
```

**Sample Input 4**

```text
empty [] brackets
archive.tar.gz
ababab
ab
```

**Sample Output 4**

```text
Module: UNKNOWN
Extension: [gz]
Count: 3
```

### Answer

```java
// UserMainCode.java
public class UserMainCode {
    public static String extractModule(String logLine) {
        int open = logLine.indexOf('[');
        if (open == -1) {
            return "UNKNOWN";
        }
        int close = logLine.indexOf(']', open + 1);
        if (close == -1 || close == open + 1) {
            return "UNKNOWN";
        }
        return logLine.substring(open + 1, close);
    }

    public static String getFileExtension(String fileName) {
        int dot = fileName.lastIndexOf('.');
        if (dot <= 0 || dot == fileName.length() - 1) {
            return "";
        }
        return fileName.substring(dot + 1).toLowerCase();
    }

    public static int countOccurrences(String text, String pattern) {
        if (pattern.isEmpty()) {
            return 0;
        }
        int count = 0;
        int from = 0;
        while ((from = text.indexOf(pattern, from)) != -1) {
            count++;
            from += pattern.length();
        }
        return count;
    }
}
```

### Reasoning

* **`indexOf(x, fromIndex)`** is the key overload. Searching for `]` starting at `open + 1` means a stray `]` **before** the `[` is ignored (Sample 3 correctly returns `X`, not `UNKNOWN`). Without the second argument, `indexOf(']')` would find the earlier bracket and `substring(open + 1, close)` would throw because `close < open`.
* **`substring(begin, end)`**: `begin` inclusive, `end` exclusive. `logLine.substring(open + 1, close)` therefore excludes both brackets. The empty-brackets case `[]` gives `close == open + 1`, an empty substring, which the requirement maps to `UNKNOWN` (Sample 4).
* **Return code `-1` is not "the last position"**: always test for `-1` before using the index in `substring`, otherwise you get `StringIndexOutOfBoundsException`.
* **`lastIndexOf('.')`** finds the last dot, so `Report.Final.PDF` gives the extension `pdf` and `archive.tar.gz` gives `gz`. Using `indexOf` would return `final.pdf` and `tar.gz`.
* **The `dot <= 0` test** handles both "no dot" (`-1`) and "hidden file" (`0`, as in `.gitignore`). `dot == length - 1` handles `file.`, where `substring(dot + 1)` would be an empty string anyway, but the explicit check states the rule clearly.
* **Non-overlapping counting**: after a match at `from`, continue at `from + pattern.length()`. In Sample 2, `aaaa` with `aa` gives `2`, not `3`. Continuing at `from + 1` would count overlaps.
* **Empty pattern guard**: `"abc".indexOf("", from)` returns `from` every time, so without the guard the loop could never finish (or would count meaninglessly). The guard is required in this method.
* **Assignment inside the loop condition** (`(from = text.indexOf(...)) != -1`) is a common idiom that updates and tests in one place. Splitting it into a `while (true)` with `break` is equally acceptable.


---

## Q8. RoadGuard Regex Validators

**Focus:** String (matches, replaceAll, groups and back-references)  ·  **Level:** Medium-Hard

RoadGuard, a vehicle registration portal for a state transport department in Karnataka, must clean and validate values entered by citizens. Write a Java program based on the following requirements. The `UserMainCode` class must contain three static methods.

**Requirement 1**

```java
public static boolean isValidVehicleNumber(String number)
```

A valid vehicle number has, with **no spaces and only uppercase letters**: 2 letters (state code), 2 digits (district), 1 or 2 letters (series), then 4 digits. Example: `TN09AB1234`, `KA01A0001`.

**Requirement 2**

```java
public static String normalizePhone(String raw)
```

Remove every character that is not a digit. Then:

* if 12 digits remain and they start with `91`, drop the `91`;
* otherwise, if 11 digits remain and they start with `0`, drop the `0`.

The result is valid only if it now has exactly **10 digits and starts with 6, 7, 8 or 9**. Return the 10 digits, or `"INVALID"`.

**Requirement 3**

```java
public static String toIsoDate(String date)
```

Convert a date from `dd/mm/yyyy` to `yyyy-mm-dd`. The input must match that format exactly (two digits, `/`, two digits, `/`, four digits); otherwise return `"INVALID DATE"`. Only the format is checked, not whether the date exists.

**Input Format**
Three lines: the vehicle number, the phone text, the date.

**Output Format**
`Vehicle valid: Yes/No`, `Phone: <result>`, `Date: <result>`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String vehicle = sc.nextLine();
        String phone = sc.nextLine();
        String date = sc.nextLine();

        System.out.println("Vehicle valid: " + (UserMainCode.isValidVehicleNumber(vehicle) ? "Yes" : "No"));
        System.out.println("Phone: " + UserMainCode.normalizePhone(phone));
        System.out.println("Date: " + UserMainCode.toIsoDate(date));
    }
}
```

**Sample Input 1**

```text
TN09AB1234
+91 98765-43210
29/09/2026
```

**Sample Output 1**

```text
Vehicle valid: Yes
Phone: 9876543210
Date: 2026-09-29
```

**Sample Input 2**

```text
tn09ab1234
12345
9/9/2026
```

**Sample Output 2**

```text
Vehicle valid: No
Phone: INVALID
Date: INVALID DATE
```

**Sample Input 3**

```text
KA01A0001
(044) 2345 6789
01/01/2027
```

**Sample Output 3**

```text
Vehicle valid: Yes
Phone: INVALID
Date: 2027-01-01
```

**Sample Input 4**

```text
TN09ABC1234
098765 43210
15-08-2026
```

**Sample Output 4**

```text
Vehicle valid: No
Phone: 9876543210
Date: INVALID DATE
```

### Answer

```java
// UserMainCode.java
public class UserMainCode {
    public static boolean isValidVehicleNumber(String number) {
        return number.matches("[A-Z]{2}[0-9]{2}[A-Z]{1,2}[0-9]{4}");
    }

    public static String normalizePhone(String raw) {
        String digits = raw.replaceAll("\\D", "");
        if (digits.length() == 12 && digits.startsWith("91")) {
            digits = digits.substring(2);
        } else if (digits.length() == 11 && digits.startsWith("0")) {
            digits = digits.substring(1);
        }
        if (digits.length() == 10 && "6789".indexOf(digits.charAt(0)) != -1) {
            return digits;
        }
        return "INVALID";
    }

    public static String toIsoDate(String date) {
        if (!date.matches("\\d{2}/\\d{2}/\\d{4}")) {
            return "INVALID DATE";
        }
        return date.replaceAll("(\\d{2})/(\\d{2})/(\\d{4})", "$3-$2-$1");
    }
}
```

### Reasoning

* **`String.matches` must match the whole string.** Unlike `Matcher.find`, it behaves as if the pattern were wrapped in `^...$`, so the vehicle pattern needs no anchors. That is why `TN09ABC1234` (three series letters, Sample 4) fails: `[A-Z]{1,2}` cannot absorb the third letter, and no partial match is accepted.
* **Regex building blocks used here**: `[A-Z]` a character class, `{2}` an exact repeat, `{1,2}` a range, `\\d` a digit, `\\D` a non-digit, `\\s+` runs of whitespace. Inside a Java string every regex backslash is doubled: `"\\d"` is the two characters `\d`.
* **`replaceAll("\\D", "")`** strips everything except digits in one call, so `+91 98765-43210` becomes `919876543210` regardless of spaces, dashes or brackets. It is the fastest way to normalise messy numeric input.
* **Apply the prefix rules to the cleaned digits only**, and check the length together with the prefix. A 10-digit number that happens to start with `91` must **not** lose those digits, which is why `length() == 12` is tested along with `startsWith("91")`.
* **`else if`** ensures only one prefix rule is applied.
* **Membership test with `indexOf`**: `"6789".indexOf(digits.charAt(0)) != -1` is a compact "first digit is one of 6-9" check. Sample 3 (`04423456789`, a landline) becomes `4423456789`, which starts with `4` and is rejected.
* **Groups and back-references**: parentheses capture parts of the match and `$1`, `$2`, `$3` reuse them in the replacement. `$3-$2-$1` reorders `dd/mm/yyyy` into `yyyy-mm-dd`. Note that the replacement string treats `$` and `\` specially, so a literal dollar sign there needs escaping.
* **Validate first, then transform**: `replaceAll` on a non-matching string returns it **unchanged**, so without the `matches` guard `9/9/2026` or `15-08-2026` would be returned as if valid.
* **`replace` vs `replaceAll`**: `replace("." , "-")` treats its argument literally, while `replaceAll(".", "-")` treats `.` as "any character" and would replace every character. Use `replace` for literal text and `replaceAll` only when you need a pattern.


---

## Q9. BillMate Invoice Formatting

**Focus:** String (String.format, replace/replaceFirst, parsing)  ·  **Level:** Medium

BillMate, an accounting-software company in Kochi, prints invoices and imports amounts typed by users. Write a Java program based on the following requirements. The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static String formatInvoiceLine(String item, int qty, double unitPrice)
```

Return one invoice line using this layout, built with `String.format`:

* the item name, **left-aligned in 12 characters** (names longer than 12 characters are cut to their first 12 characters),
* the quantity, **right-aligned in 4 characters**,
* the text ` x `,
* the unit price, right-aligned in 8 characters with **2 decimals**,
* the text ` = `,
* the line total (`qty * unitPrice`), right-aligned in 10 characters with **2 decimals**.

**Requirement 2**

```java
public static double parseAmount(String text)
```

Convert text typed by a user into a number. The text may contain **commas, spaces, and a currency prefix** made of non-digit characters (for example `Rs.1,250.50` or `$99`). After removing these, what remains must be a plain number: digits with an optional single decimal part (`123` or `123.45`). Return that value, or `-1` if the text is not a valid amount.

**Input Format**
Line 1: `item qty unitPrice` (the item has no spaces). Line 2: the amount text.

**Output Format**
The invoice line between `|` characters (so the padding is visible), then `Parsed: <value>`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String item = sc.next();
        int qty = sc.nextInt();
        double price = sc.nextDouble();
        sc.nextLine();
        String amountText = sc.nextLine();

        System.out.println("|" + UserMainCode.formatInvoiceLine(item, qty, price) + "|");
        System.out.println("Parsed: " + UserMainCode.parseAmount(amountText));
    }
}
```

**Sample Input 1**

```text
Notebook 5 45.5
Rs.1,250.50
```

**Sample Output 1**

```text
|Notebook       5 x    45.50 =     227.50|
Parsed: 1250.5
```

**Sample Input 2**

```text
Whiteboard-Marker-Set 12 7.25
$99
```

**Sample Output 2**

```text
|Whiteboard-M  12 x     7.25 =      87.00|
Parsed: 99.0
```

**Sample Input 3**

```text
Pen 1 10
12.5.3
```

**Sample Output 3**

```text
|Pen            1 x    10.00 =      10.00|
Parsed: -1.0
```

**Sample Input 4**

```text
Stapler 100 199.999
  Rs 2,00,000  
```

**Sample Output 4**

```text
|Stapler      100 x   200.00 =   19999.90|
Parsed: 200000.0
```

### Answer

```java
// UserMainCode.java
public class UserMainCode {
    public static String formatInvoiceLine(String item, int qty, double unitPrice) {
        String name = item.length() > 12 ? item.substring(0, 12) : item;
        return String.format("%-12s%4d x %8.2f = %10.2f", name, qty, unitPrice, qty * unitPrice);
    }

    public static double parseAmount(String text) {
        String cleaned = text.replace(",", "")
                             .replace(" ", "")
                             .replaceFirst("^[^0-9]+", "");
        if (!cleaned.matches("\\d+(\\.\\d+)?")) {
            return -1;
        }
        return Double.parseDouble(cleaned);
    }
}
```

### Reasoning

* **Format specifiers**: `%s` string, `%d` integer, `%f` floating point. Between `%` and the letter, a number sets the **minimum width** and `-` means **left-align**. `%-12s` is left-aligned in 12 columns, `%4d` is right-aligned in 4, and `%8.2f` is 8 wide with 2 decimals. The default alignment is right, which is right for numbers.
* **Argument order and count must match the specifiers.** A missing argument throws `MissingFormatArgumentException`, and `%d` given a `double` throws `IllegalFormatConversionException`. Note that `qty * unitPrice` is a `double` because of the `double` operand, so it pairs with `%f`.
* **Widths are minimums**: a longer value is never truncated by `%12s`. That is why the item is cut with `substring(0, 12)` beforehand (Sample 2). `%.12s` is an alternative that truncates inside the format itself.
* **`%.2f` rounds** (it does not truncate). Sample 4's unit price `199.999` prints as `200.00`, and the total `19999.90` also shows two decimals. Java's `String.format` also uses the **default locale**, which can print a decimal comma. For guaranteed output use `String.format(Locale.US, ...)`.
* **Parsing pipeline**: remove commas, remove spaces, remove the non-digit prefix, validate, then parse. `replaceFirst("^[^0-9]+", "")` removes only a leading run of non-digits, so `Rs.1250.50` correctly keeps its decimal point. A simpler `replaceAll("[^0-9.]", "")` would leave `.` from `Rs.` at the front and turn `Rs.500` into `.500`, which parses as `0.5`.
* **Validate with `matches` before `parseDouble`.** `Double.parseDouble` is more permissive than it looks: it accepts `"5d"`, `"1e3"`, `"NaN"` and `"Infinity"`. The pattern `\d+(\.\d+)?` allows only plain numbers, and `12.5.3` (Sample 3) is rejected without any exception. The other valid approach is `try { ... } catch (NumberFormatException e) { return -1; }`.
* **Indian-style grouping** (`2,00,000` in Sample 4) works because every comma is removed regardless of position.
* **After `nextDouble()`, call `nextLine()`** to consume the rest of the line before reading the amount line. Forgetting it makes the driver read an empty line as the amount.


---


---

# Part B: StringBuilder and StringBuffer

## Q10. PackRight Serial Code Formatter

**Focus:** StringBuilder (insert, append, reverse, setCharAt)  ·  **Level:** Easy-Medium

PackRight Logistics, a packaging company in Ahmedabad, prints serial codes and card details on shipping labels. Write a Java program based on the following requirements. The `UserMainCode` class must contain three static methods.

**Requirement 1**

```java
public static String groupCode(String raw, int groupSize)
```

Remove all hyphens and spaces from `raw`, convert it to **uppercase**, and then insert a hyphen after every `groupSize` characters. There must be **no hyphen at the end**, even when the length is an exact multiple of `groupSize`. `groupSize` is at least 1.

**Requirement 2**

```java
public static String reverseEachWord(String sentence)
```

Reverse the letters of **each word** while keeping the word order. Words are separated by exactly one space. An empty sentence returns an empty string.

**Requirement 3**

```java
public static String maskCard(String cardNumber)
```

Replace every character except the **last four** with `*`. If the card number has four or fewer characters, return it unchanged.

**Input Format**
Four lines: the raw code, the group size, the sentence, the card number.

**Output Format**
`Code: <result>`, `Words: <result>`, `Card: <result>`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String raw = sc.nextLine();
        int size = Integer.parseInt(sc.nextLine().trim());
        String sentence = sc.nextLine();
        String card = sc.nextLine();

        System.out.println("Code: " + UserMainCode.groupCode(raw, size));
        System.out.println("Words: " + UserMainCode.reverseEachWord(sentence));
        System.out.println("Card: " + UserMainCode.maskCard(card));
    }
}
```

**Sample Input 1**

```text
ab12-cd 34ef5
4
hello big world
4111111111111234
```

**Sample Output 1**

```text
Code: AB12-CD34-EF5
Words: olleh gib dlrow
Card: ************1234
```

**Sample Input 2**

```text
abc
5
a
12
```

**Sample Output 2**

```text
Code: ABC
Words: a
Card: 12
```

**Sample Input 3**

```text
abcdefgh
4
go
12345
```

**Sample Output 3**

```text
Code: ABCD-EFGH
Words: og
Card: *2345
```

**Sample Input 4**

```text
x1y2z3
1
Madam Arora teaches malayalam
1234
```

**Sample Output 4**

```text
Code: X-1-Y-2-Z-3
Words: madaM arorA sehcaet malayalam
Card: 1234
```

### Answer

```java
// UserMainCode.java
public class UserMainCode {
    public static String groupCode(String raw, int groupSize) {
        String cleaned = raw.replace("-", "").replace(" ", "").toUpperCase();
        StringBuilder sb = new StringBuilder(cleaned);
        for (int i = groupSize; i < sb.length(); i += groupSize + 1) {
            sb.insert(i, '-');
        }
        return sb.toString();
    }

    public static String reverseEachWord(String sentence) {
        if (sentence.isEmpty()) {
            return "";
        }
        String[] words = sentence.split(" ");
        StringBuilder result = new StringBuilder();
        for (int i = 0; i < words.length; i++) {
            if (i > 0) {
                result.append(' ');
            }
            result.append(new StringBuilder(words[i]).reverse());
        }
        return result.toString();
    }

    public static String maskCard(String cardNumber) {
        int maskLength = cardNumber.length() - 4;
        if (maskLength <= 0) {
            return cardNumber;
        }
        StringBuilder sb = new StringBuilder(cardNumber);
        for (int i = 0; i < maskLength; i++) {
            sb.setCharAt(i, '*');
        }
        return sb.toString();
    }
}
```

### Reasoning

* **`insert(offset, x)` shifts everything after the offset to the right**, so after inserting a hyphen the next insertion point moves by `groupSize + 1`, not `groupSize`. That is why the loop steps `i += groupSize + 1`. The loop condition `i < sb.length()` (checked on the **growing** builder) is what prevents a trailing hyphen: for `ABCDEFGH` with size 4, after inserting at 4 the next index is 9, which is not less than the length 9 (Sample 3).
* **Alternative approach** (equally valid, often easier to reason about): append characters one by one and put a hyphen before the character when `i > 0 && i % groupSize == 0`. Insert-based code is more compact but easier to get off by one.
* **Chain `replace` calls** for cleaning: `replace("-", "").replace(" ", "")`. Each call returns a new `String`, so the chain is needed. `toUpperCase()` last.
* **`new StringBuilder(word).reverse()`** is the standard one-liner for reversing a string, as `String` has no `reverse()`. `reverse()` returns the same builder, so it can be passed straight to `append`. Sample 4 shows why palindromic words (`Madam`, `malayalam`) need care: `Madam` reversed is `madaM`, not `Madam`, because case is preserved.
* **`sb.append(otherBuilder)`** works because `StringBuilder.append(CharSequence)` accepts another builder. `toString()` is not required but is harmless.
* **`setCharAt(index, char)`** overwrites in place and is the way to mutate a single character. `String` has no equivalent. The index must be `< length()`, otherwise `StringIndexOutOfBoundsException`. Another option is `sb.replace(0, maskLength, stars)` with a prebuilt run of `*`.
* **Off-by-one on the mask**: `length - 4` characters are masked, and the guard `<= 0` covers the short card (Sample 2) as well as the exact four-digit case.
* **`split(" ")` vs `split("\\s+")`**: the requirement guarantees a single space, so a plain `" "` is enough here. On messy input it would produce empty words.


---

## Q11. DocuClean Text Cleaner

**Focus:** StringBuilder (deleteCharAt, delete, indexOf, charAt, setCharAt)  ·  **Level:** Medium

DocuClean, a document-processing firm in Gurugram, cleans up noisy text before archiving it. Write a Java program based on the following requirements. The `UserMainCode` class must contain three static methods. **Use `StringBuilder`, and do not use `String.replace`, `replaceAll` or regular expressions.**

**Requirement 1**

```java
public static String removeConsecutiveDuplicates(String text)
```

Collapse every run of the **same character** into a single character. Comparison is case-sensitive. Example: `aabbbcaa` becomes `abca`.

**Requirement 2**

```java
public static String removeAllOccurrences(String text, String word)
```

Remove every occurrence of `word` from `text`, scanning left to right in a single pass (case-sensitive). If `word` is empty, return `text` unchanged.

**Requirement 3**

```java
public static String toggleCase(String text)
```

Swap the case of every letter (uppercase becomes lowercase and vice versa). Characters that are not letters stay as they are.

**Input Format**
Line 1: the text. Line 2: the word to remove (it may be an empty line).

**Output Format**
Three lines: `Deduped: [<result>]`, `Removed: [<result>]`, `Toggled: [<result>]`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String text = sc.nextLine();
        String word = sc.nextLine();

        System.out.println("Deduped: [" + UserMainCode.removeConsecutiveDuplicates(text) + "]");
        System.out.println("Removed: [" + UserMainCode.removeAllOccurrences(text, word) + "]");
        System.out.println("Toggled: [" + UserMainCode.toggleCase(text) + "]");
    }
}
```

**Sample Input 1**

```text
Good Goood GOOD
oo
```

**Sample Output 1**

```text
Deduped: [God God GOD]
Removed: [Gd God GOOD]
Toggled: [gOOD gOOOD good]
```

**Sample Input 2**

```text
aaa
aa
```

**Sample Output 2**

```text
Deduped: [a]
Removed: [a]
Toggled: [AAA]
```

**Sample Input 3**

```text
abbcc
bc
```

**Sample Output 3**

```text
Deduped: [abc]
Removed: [abc]
Toggled: [ABBCC]
```

**Sample Input 4**

```text
Hello  World

```

**Sample Output 4**

```text
Deduped: [Helo World]
Removed: [Hello  World]
Toggled: [hELLO  wORLD]
```

### Answer

```java
// UserMainCode.java
public class UserMainCode {
    public static String removeConsecutiveDuplicates(String text) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < text.length(); i++) {
            char c = text.charAt(i);
            if (sb.length() == 0 || sb.charAt(sb.length() - 1) != c) {
                sb.append(c);
            }
        }
        return sb.toString();
    }

    public static String removeAllOccurrences(String text, String word) {
        if (word.isEmpty()) {
            return text;
        }
        StringBuilder sb = new StringBuilder(text);
        int idx = sb.indexOf(word);
        while (idx != -1) {
            sb.delete(idx, idx + word.length());
            idx = sb.indexOf(word, idx);
        }
        return sb.toString();
    }

    public static String toggleCase(String text) {
        StringBuilder sb = new StringBuilder(text);
        for (int i = 0; i < sb.length(); i++) {
            char c = sb.charAt(i);
            if (Character.isUpperCase(c)) {
                sb.setCharAt(i, Character.toLowerCase(c));
            } else if (Character.isLowerCase(c)) {
                sb.setCharAt(i, Character.toUpperCase(c));
            }
        }
        return sb.toString();
    }
}
```

### Reasoning

* **Compare with the last character kept, not with the previous input character.** `sb.charAt(sb.length() - 1)` is the last character already in the result. The `sb.length() == 0` check must come first so the very first character is accepted without calling `charAt(-1)`.
* **`delete(start, end)`** removes the range `[start, end)`: `start` inclusive, `end` exclusive. It shifts the remaining text left and shortens the builder. Contrast with `deleteCharAt(index)`, which removes a single character (an alternative for Requirement 1 that works backwards through the text).
* **Restart the search at `idx`, not at `0` or `idx + word.length()`.** After deleting, the text that followed the removed word now starts at `idx`, so searching from `idx` continues the left-to-right pass without skipping anything. Searching from `0` each time would rescan text already checked and could also join characters across the gap into a new match, producing a different result from a single pass. In Sample 3 (`abbcc` remove `bc`), one pass gives `abc`, because the `b` before the gap and the `c` after it are not re-matched.
* **`StringBuilder.indexOf(String, fromIndex)`** works like the `String` version. The empty-word guard is essential: `indexOf("")` always succeeds and `delete(idx, idx)` removes nothing, so the loop would never end (the last sample has an **empty second line**).
* **Overlapping removals** (Sample 2): `aaa` remove `aa` gives `a`, matching what `String.replace` would do. The first `aa` is removed and the leftover `a` is not a match.
* **Why the builder beats repeated `String` operations**: deleting inside a `String` means building a whole new string each time. A builder edits one buffer in place.
* **`setCharAt` needs a real index**: the loop uses `sb.length()` (not `text.length()`), which is the same value here because case toggling never changes the length. Non-letters such as digits and spaces fall through both branches and stay unchanged. `Character.toUpperCase` on a non-letter also returns it unchanged, but the explicit checks make intent clear.
* **Java 8 note**: `sb.compareTo(other)` and `sb.isEmpty()` do not exist before Java 11 and 15 respectively, so `sb.length() == 0` is the portable emptiness test.


---

## Q12. AuditTrail Report Builder

**Focus:** StringBuilder (setLength, length, equals trap, in-place editing)  ·  **Level:** Medium-Hard

AuditTrail Systems, a compliance-software company in Mumbai, assembles text reports from log lines. Write a Java program based on the following requirements. The `UserMainCode` class must contain three static methods.

**Requirement 1**

```java
public static String buildReport(String[] lines, int maxLength)
```

Join the lines with a **newline character** (`\n`) between them (no newline after the last line). If the joined text is **longer than `maxLength`**, cut it so that the result is exactly `maxLength` characters long and **ends with `...`** (the three dots are part of the `maxLength`). `maxLength` is at least 3. An empty array returns an empty string.

**Requirement 2**

```java
public static boolean isSameContent(StringBuilder a, StringBuilder b)
```

Return `true` if the two builders contain the same characters.

**Requirement 3**

```java
public static StringBuilder trimTrailingSpaces(StringBuilder sb)
```

Remove all trailing spaces from the given builder **in place** and return the same builder.

**Input Format**
`n`, then `n` lines of report text, then `maxLength`, then three lines: the first builder's text, the second builder's text, and the text to trim.

**Output Format**
Prints the report (newlines shown as `\n`) with its length, whether the builders have the same content, what `equals` says, and the trimmed text with its length.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = Integer.parseInt(sc.nextLine().trim());
        String[] lines = new String[n];
        for (int i = 0; i < n; i++) {
            lines[i] = sc.nextLine();
        }
        int maxLength = Integer.parseInt(sc.nextLine().trim());
        StringBuilder a = new StringBuilder(sc.nextLine());
        StringBuilder b = new StringBuilder(sc.nextLine());
        StringBuilder toTrim = new StringBuilder(sc.nextLine());

        String report = UserMainCode.buildReport(lines, maxLength);
        System.out.println("Report: [" + report.replace("\n", "\\n") + "]");
        System.out.println("Length: " + report.length());
        System.out.println("Same content: " + (UserMainCode.isSameContent(a, b) ? "Yes" : "No"));
        System.out.println("equals() says: " + a.equals(b));

        StringBuilder trimmed = UserMainCode.trimTrailingSpaces(toTrim);
        System.out.println("Trimmed: [" + trimmed + "]");
        System.out.println("Same object returned: " + (trimmed == toTrim));
        System.out.println("Trimmed length: " + trimmed.length());
    }
}
```

**Sample Input 1**

```text
3
Sales up 10%
Costs down 5%
Net positive
30
hello
hello
hello   
```

**Sample Output 1**

```text
Report: [Sales up 10%\nCosts down 5%\n...]
Length: 30
Same content: Yes
equals() says: false
Trimmed: [hello]
Same object returned: true
Trimmed length: 5
```

**Sample Input 2**

```text
1
Short
100
abc
abd
  x  
```

**Sample Output 2**

```text
Report: [Short]
Length: 5
Same content: No
equals() says: false
Trimmed: [  x]
Same object returned: true
Trimmed length: 3
```

**Sample Input 3**

```text
2
ab
cd
5
Same
Same
    
```

**Sample Output 3**

```text
Report: [ab\ncd]
Length: 5
Same content: Yes
equals() says: false
Trimmed: []
Same object returned: true
Trimmed length: 0
```

### Answer

```java
// UserMainCode.java
public class UserMainCode {
    public static String buildReport(String[] lines, int maxLength) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < lines.length; i++) {
            if (i > 0) {
                sb.append('\n');
            }
            sb.append(lines[i]);
        }
        if (sb.length() > maxLength) {
            sb.setLength(maxLength - 3);
            sb.append("...");
        }
        return sb.toString();
    }

    public static boolean isSameContent(StringBuilder a, StringBuilder b) {
        return a.toString().equals(b.toString());
    }

    public static StringBuilder trimTrailingSpaces(StringBuilder sb) {
        int len = sb.length();
        while (len > 0 && sb.charAt(len - 1) == ' ') {
            len--;
        }
        sb.setLength(len);
        return sb;
    }
}
```

### Reasoning

* **`StringBuilder` does not override `equals`.** `a.equals(b)` is inherited from `Object` and compares **references**, so two builders holding `hello` are "not equal" (the driver prints `false`). The fix is to compare the contents: `a.toString().equals(b.toString())`. `String.contentEquals(CharSequence)` (`a.toString().contentEquals(b)`) and, on Java 11+, `a.compareTo(b) == 0` are alternatives. The same trap applies to `StringBuffer`.
* **`setLength(n)` truncates or pads.** Setting it smaller cuts the builder to the first `n` characters, which is the cheapest way to "drop the tail". Setting it larger pads with `'\u0000'` characters, so never use it to extend text. `setLength(0)` clears a builder for reuse.
* **Truncation arithmetic**: to end up with exactly `maxLength` characters including `...`, keep `maxLength - 3` characters and append the three dots. In Sample 1 the joined text is 39 characters, the limit is 30, so the first 27 characters are kept. Note that the newline **counts as one character**, so the cut lands just after the second `\n`.
* **Only truncate when needed**: the condition is `> maxLength`, not `>=`. Text of exactly `maxLength` characters must be left intact (Sample 3: `ab\ncd` is 5 characters and passes through).
* **In-place edit and returning the same object**: `trimTrailingSpaces` changes the builder it is given and returns that same reference (the driver prints `Same object returned: true`). This is the pattern behind method chaining, where `append`, `insert` and `reverse` all return `this`.
* **Trim by scanning from the end** and only then call `setLength` once. Deleting one space at a time with `deleteCharAt(length - 1)` is also correct but does more work. A `String`'s `trim()` cannot be used because the value here is a builder (and `StringBuilder` has no `trim()` method).
* **`length()` vs `capacity()`**: `length()` is the number of characters currently held, `capacity()` is the size of the internal buffer. `setLength` changes the length, and capacity grows automatically when needed. `trimToSize()` releases unused capacity.
* **Empty input array** returns `""` because the loop never runs and the length is `0`, which is not greater than `maxLength`.


---

## Q13. TicketDesk Shared Log (StringBuffer)

**Focus:** StringBuffer (thread safety)  ·  **Level:** Medium-Hard

TicketDesk Solutions, a customer-support platform in Pune, has several counter threads that write messages into one shared log at the same time. Nothing may be lost or corrupted, even when many threads write simultaneously. Develop a Java program based on the following requirements.

**Requirement 1**
Create a `LogBuffer` class containing a `StringBuffer` named `buffer`.
Implement:

```java
public void log(String message)
```

Append the message followed by a newline character (`\n`) to `buffer`.

**Requirement 2**
Implement:

```java
public int getLineCount()
```

Return the number of lines logged so far (the number of newline characters in the log).

**Requirement 3**
Implement:

```java
public String getLog()
```

Return the complete log text.

**Requirement 4**
Implement:

```java
public void clear()
```

Remove all content from the log (without creating a new `StringBuffer`).

**Restrictions**

* Edit only the `LogBuffer` class.
* The buffer must be a `StringBuffer` (not a `StringBuilder`).
* Do **not** use `synchronized` blocks, `Lock` objects or any other explicit locking.
* Attributes must be `private`. Constructor and methods must be `public`.
* Do not change the specified class, attribute, or method names.
* Do not use `System.exit(0)`.

**Input Format**
Two integers: the number of threads and the number of messages each thread logs.

**Given driver code** (already provided; do not modify)

```java
// Main.java
import java.util.*;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        Scanner sc = new Scanner(System.in);
        int threads = sc.nextInt();
        int perThread = sc.nextInt();

        LogBuffer log = new LogBuffer();
        Thread[] workers = new Thread[threads];
        for (int t = 0; t < threads; t++) {
            final int id = t + 1;
            workers[t] = new Thread(() -> {
                for (int i = 0; i < perThread; i++) {
                    log.log("T" + id + "-" + i);
                }
            });
            workers[t].start();
        }
        for (Thread w : workers) {
            w.join();
        }

        System.out.println("Lines logged: " + log.getLineCount());
        System.out.println("Log length: " + log.getLog().length());

        log.clear();
        System.out.println("After clear: [" + log.getLog() + "] lines=" + log.getLineCount());
    }
}
```

**Sample Input 1**

```text
4 1000
```

**Sample Output 1**

```text
Lines logged: 4000
Log length: 27560
After clear: [] lines=0
```

**Sample Input 2**

```text
2 5
```

**Sample Output 2**

```text
Lines logged: 10
Log length: 50
After clear: [] lines=0
```

**Sample Input 3**

```text
1 3
```

**Sample Output 3**

```text
Lines logged: 3
Log length: 15
After clear: [] lines=0
```

### Answer

```java
// LogBuffer.java
public class LogBuffer {
    private StringBuffer buffer;

    public LogBuffer() {
        this.buffer = new StringBuffer();
    }

    public void log(String message) {
        buffer.append(message + "\n");
    }

    public int getLineCount() {
        String text = buffer.toString();
        int count = 0;
        for (int i = 0; i < text.length(); i++) {
            if (text.charAt(i) == '\n') {
                count++;
            }
        }
        return count;
    }

    public String getLog() {
        return buffer.toString();
    }

    public void clear() {
        buffer.setLength(0);
    }
}
```

### Reasoning

* **`StringBuffer` = `StringBuilder` + synchronisation.** Every public method of `StringBuffer` (`append`, `insert`, `delete`, `reverse`, `setLength`, and so on) is `synchronized`, so two threads can never be inside the buffer at the same time. `StringBuilder` has the same API without the locking, which makes it faster for single-threaded code and unsafe to share between threads.
* **Why the counts are exact**: with four threads each logging 1000 messages, the line count is always `4000` and the total length is always the same, because no `append` can overwrite another. With a shared `StringBuilder`, concurrent appends can overwrite each other's characters or throw `ArrayIndexOutOfBoundsException` while the internal array is being resized. In a test run of the same driver with `StringBuilder` and 4 threads x 20000 messages, the line count came out at values such as 62491 and 79998 instead of 80000 (and one run crashed), even though the reported **length** was still correct. The length counter is updated separately from the characters, so a corrupted log can still have the right length. That is exactly why this driver counts newlines. The outcome is timing-dependent, which makes it a nasty bug: a test may pass on some runs.
* **Compound operations are not atomic.** Each single call is safe, but `buffer.append(message).append("\n")` is **two** calls, so another thread's text can land between them and glue two messages onto one line. The solution builds `message + "\n"` first and performs **one** `append`. If a method must perform several buffer calls as a unit, the method itself needs to be `synchronized`. The driver here would not detect the interleaving (the counts stay right), but a real log would be corrupted.
* **Thread safety is per call, not per program logic.** Reading with `getLog()` while other threads still write returns a consistent snapshot of a moment in time, not a "final" report. That is why the driver calls `join()` on every thread before printing.
* **`setLength(0)` clears in place** and keeps the same object (and its capacity). Creating `new StringBuffer()` in `clear()` would also work for a single thread, but it would change the object other code may be holding.
* **When to choose which**: use `StringBuilder` by default (local variables, single thread). Use `StringBuffer` only when the same buffer is genuinely shared by multiple threads and you want per-call safety without writing locks. In modern code `StringBuilder` plus explicit synchronisation, or a thread-safe queue of messages, is often preferred, but assessments still test the distinction.
* **Counting newlines**: `getLineCount` converts to a `String` once and scans it, so the count is taken from one consistent snapshot. Calling `buffer.charAt(i)` in a loop would take the lock on every iteration and could see the buffer change mid-scan.


---

## Q14. ArchiveX Run-Length Compression

**Focus:** StringBuilder (append(char), append(int), parsing digits)  ·  **Level:** Hard

ArchiveX, a data-archival company in Bengaluru, stores repetitive text in a compact run-length format. Write a Java program based on the following requirements. The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static String compress(String text)
```

Replace every run of the same character by the character followed by the **length of the run**, **always** writing the count, even when it is 1. Case-sensitive. Example: `aaabbc` becomes `a3b2c1`, and `AAaa` becomes `A2a2`. The input contains only letters. An empty string returns an empty string.

**Requirement 2**

```java
public static String decompress(String encoded)
```

Reverse the operation. The input is always well formed: a letter, followed by a count of **one or more digits** (for example `b12` means twelve `b` characters), repeated. An empty string returns an empty string.

**Input Format**
Line 1: the text to compress (may be empty). Line 2: the encoded text to decompress (may be empty).

**Output Format**
`Compressed: [<result>]` then `Decompressed: [<result>]`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String text = sc.nextLine();
        String encoded = sc.nextLine();
        System.out.println("Compressed: [" + UserMainCode.compress(text) + "]");
        System.out.println("Decompressed: [" + UserMainCode.decompress(encoded) + "]");
    }
}
```

**Sample Input 1**

```text
aaabbbbbbbbbbbbc
x2Y3z10
```

**Sample Output 1**

```text
Compressed: [a3b12c1]
Decompressed: [xxYYYzzzzzzzzzz]
```

**Sample Input 2**

```text

a1
```

**Sample Output 2**

```text
Compressed: []
Decompressed: [a]
```

**Sample Input 3**

```text
AAaa
b1c1
```

**Sample Output 3**

```text
Compressed: [A2a2]
Decompressed: [bc]
```

**Sample Input 4**

```text
zzzzzzzzzzzzzzzzzzzzzzzzz
q0r2
```

**Sample Output 4**

```text
Compressed: [z25]
Decompressed: [rr]
```

### Answer

```java
// UserMainCode.java
public class UserMainCode {
    public static String compress(String text) {
        if (text.isEmpty()) {
            return "";
        }
        StringBuilder sb = new StringBuilder();
        char current = text.charAt(0);
        int count = 1;

        for (int i = 1; i < text.length(); i++) {
            char c = text.charAt(i);
            if (c == current) {
                count++;
            } else {
                sb.append(current).append(count);
                current = c;
                count = 1;
            }
        }
        sb.append(current).append(count);   // flush the final run
        return sb.toString();
    }

    public static String decompress(String encoded) {
        StringBuilder sb = new StringBuilder();
        int i = 0;
        while (i < encoded.length()) {
            char letter = encoded.charAt(i++);
            int start = i;
            while (i < encoded.length() && Character.isDigit(encoded.charAt(i))) {
                i++;
            }
            int count = Integer.parseInt(encoded.substring(start, i));
            for (int k = 0; k < count; k++) {
                sb.append(letter);
            }
        }
        return sb.toString();
    }
}
```

### Reasoning

* **The "flush the last run" pattern.** A run is written only when the character *changes*, so the final run is still pending when the loop ends. The `append` after the loop writes it. Forgetting it drops the last group (`aaabbc` would give `a3b2`), which is the classic bug in this problem.
* **`append(char)` and `append(int)` are separate overloads.** `sb.append(current).append(count)` writes `a` then `3`. The tempting `sb.append(current + count)` adds the character code and the count as **integers** (97 + 3) and appends `100`.
* **Multi-digit counts** (`b12`) need a digit loop in `decompress`. Reading a single digit would produce one `b` and then treat `2` as a letter. The two-index technique (`start`, then advance `i` while `isDigit`) isolates the whole number so `Integer.parseInt(encoded.substring(start, i))` can convert it.
* **Why decode with `Character.isDigit` rather than `letter != ...`**: the format guarantees a letter is followed by digits, so digits mark the end of a count reliably. It would stop working if the original text could contain digits, which is why the requirement limits the input to letters.
* **`charAt(i++)` reads and advances in one step** (post-increment). It is neat but easy to misuse, so make sure `start = i` is captured **after** the increment, as here.
* **Compression is not always shorter.** Because counts are always written, `abc` becomes `a1b1c1`. Real compressors skip counts of 1; this question does not, to keep the format unambiguous.
* **Case sensitivity** (Sample 3): `AAaa` gives `A2a2` because `'A'` and `'a'` are different characters, so there are two runs, not one.
* **Zero count** (Sample 4, `q0`): the loop `for (k < count)` simply appends nothing, so `q0` contributes no characters while `r2` gives `rr`.
* **Performance**: one builder, one pass, `O(n)` time. Building the result by `result += ...` in a loop would create a new `String` per step.


---

## Q15. CipherDesk Caesar Cipher and Rotation

**Focus:** StringBuilder + String (char arithmetic, contains)  ·  **Level:** Medium

CipherDesk, a cyber-security training company in Chennai, is preparing beginner cryptography exercises. Write a Java program based on the following requirements. The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static String caesarCipher(String text, int shift)
```

Shift every **letter** of the text by `shift` positions in the alphabet, wrapping around from `z` to `a` (and `Z` to `A`). Uppercase letters stay uppercase and lowercase stay lowercase. Characters that are not letters (digits, spaces, punctuation) remain unchanged. `shift` may be **negative** or larger than 26.

**Requirement 2**

```java
public static boolean isRotation(String a, String b)
```

Return `true` if `b` can be obtained by rotating `a` (moving some number of characters from the front to the back). For example `erbottlewat` is a rotation of `waterbottle`. The check is case-sensitive. Do **not** try every rotation in a loop: solve it with a single `contains` call.

**Input Format**
Line 1: the text. Line 2: the shift. Line 3: string `a`. Line 4: string `b`.

**Output Format**
`Encrypted: [<result>]` then `Rotation: Yes` or `Rotation: No`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String text = sc.nextLine();
        int shift = Integer.parseInt(sc.nextLine().trim());
        String a = sc.nextLine();
        String b = sc.nextLine();

        System.out.println("Encrypted: [" + UserMainCode.caesarCipher(text, shift) + "]");
        System.out.println("Rotation: " + (UserMainCode.isRotation(a, b) ? "Yes" : "No"));
    }
}
```

**Sample Input 1**

```text
Hello, World!
3
waterbottle
erbottlewat
```

**Sample Output 1**

```text
Encrypted: [Khoor, Zruog!]
Rotation: Yes
```

**Sample Input 2**

```text
Khoor, Zruog!
-3
abc
acb
```

**Sample Output 2**

```text
Encrypted: [Hello, World!]
Rotation: No
```

**Sample Input 3**

```text
xyz ABC 123
-29
aa
aaa
```

**Sample Output 3**

```text
Encrypted: [uvw XYZ 123]
Rotation: No
```

**Sample Input 4**

```text
Java 21
55
abab
baba
```

**Sample Output 4**

```text
Encrypted: [Mdyd 21]
Rotation: Yes
```

### Answer

```java
// UserMainCode.java
public class UserMainCode {
    public static String caesarCipher(String text, int shift) {
        int s = Math.floorMod(shift, 26);
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < text.length(); i++) {
            char c = text.charAt(i);
            if (c >= 'A' && c <= 'Z') {
                sb.append((char) ('A' + (c - 'A' + s) % 26));
            } else if (c >= 'a' && c <= 'z') {
                sb.append((char) ('a' + (c - 'a' + s) % 26));
            } else {
                sb.append(c);
            }
        }
        return sb.toString();
    }

    public static boolean isRotation(String a, String b) {
        return a.length() == b.length() && (a + a).contains(b);
    }
}
```

### Reasoning

* **Characters are numbers.** `c - 'a'` converts a lowercase letter to `0..25`, adding the shift and taking `% 26` wraps it, and adding `'a'` converts back. The final `(char)` cast is required: `'a' + int` is an `int`, and appending an `int` to a `StringBuilder` would write digits such as `100` instead of the letter.
* **Normalise the shift with `Math.floorMod(shift, 26)`.** In Java the `%` operator keeps the sign of the dividend, so `-3 % 26` is `-3`, and `'a' + (c - 'a' - 3) % 26` can fall below `'a'` and produce garbage. `floorMod` always returns `0..25`: `floorMod(-29, 26)` is `23` (Sample 3) and `floorMod(55, 26)` is `3` (Sample 4). Reducing `shift` first also makes very large values safe.
* **Range checks vs `Character.isUpperCase`**: for this cipher, `c >= 'A' && c <= 'Z'` is deliberate. `isUpperCase` is also true for letters like `É`, and the modulo-26 arithmetic would be wrong for them. Anything outside `A-Z` and `a-z`, including digits and accented letters, is passed through unchanged.
* **Case preservation** is achieved by using separate branches with different base characters (`'A'` and `'a'`) rather than lowercasing.
* **Decrypting** is the same method with the negative shift: `caesarCipher("Khoor, Zruog!", -3)` returns `Hello, World!` (Sample 2).
* **Rotation trick**: every rotation of `a` is a substring of `a + a`. For `waterbottle`, `waterbottlewaterbottle` contains `erbottlewat`. The **length check must come first**: for `a = "aa"` and `b = "aaa"`, `a + a` is `aaaa`, which contains `aaa`, so without the length check the answer would wrongly be `true` (Sample 3).
* **Same length but not a rotation**: `abc` and `acb` (Sample 2) fail because `abcabc` does not contain `acb`. `abab` and `baba` (Sample 4) succeed.
* **Building with a `StringBuilder`** keeps the loop linear. The empty text case simply returns an empty string.


---

## Q16. SearchLab Longest Word and Unique Window

**Focus:** String + StringBuilder (split with regex, sliding window)  ·  **Level:** Hard

SearchLab, a search-engine start-up in Hyderabad, is prototyping text-analysis features. Write a Java program based on the following requirements. The `UserMainCode` class must contain two static methods.

**Requirement 1**

```java
public static String findLongestWord(String sentence)
```

A **word** is a maximal run of English letters (`A-Z`, `a-z`). Every other character, such as spaces, digits, apostrophes and hyphens, separates words. Return the **longest** word. If several words have the same maximum length, return the **first** one. If there are no words, return an empty string.

**Requirement 2**

```java
public static String longestUniqueSubstring(String s)
```

Return the longest substring of `s` in which **no character repeats** (case-sensitive). If several substrings share the maximum length, return the one that **starts first**. An empty string returns an empty string. Use a `StringBuilder` as the sliding window.

**Input Format**
Line 1: the sentence. Line 2: the string `s` (it may be empty).

**Output Format**
`Longest word: [<result>]` then `Longest unique: [<result>]`.

**Given driver code** (already provided; do not modify)

```java
// UserInterface.java
import java.util.*;

public class UserInterface {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        String sentence = sc.nextLine();
        String s = sc.nextLine();

        System.out.println("Longest word: [" + UserMainCode.findLongestWord(sentence) + "]");
        System.out.println("Longest unique: [" + UserMainCode.longestUniqueSubstring(s) + "]");
    }
}
```

**Sample Input 1**

```text
The quick, brown fox jumped!
abcabcbb
```

**Sample Output 1**

```text
Longest word: [jumped]
Longest unique: [abc]
```

**Sample Input 2**

```text
It's a well-known fact
pwwkew
```

**Sample Output 2**

```text
Longest word: [known]
Longest unique: [wke]
```

**Sample Input 3**

```text
??! 123
dvdf
```

**Sample Output 3**

```text
Longest word: []
Longest unique: [vdf]
```

**Sample Input 4**

```text
cat bat rat
abba
```

**Sample Output 4**

```text
Longest word: [cat]
Longest unique: [ab]
```

### Answer

```java
// UserMainCode.java
public class UserMainCode {
    public static String findLongestWord(String sentence) {
        String[] tokens = sentence.split("[^A-Za-z]+");
        String best = "";
        for (String t : tokens) {
            if (t.length() > best.length()) {
                best = t;
            }
        }
        return best;
    }

    public static String longestUniqueSubstring(String s) {
        StringBuilder window = new StringBuilder();
        String best = "";

        for (int i = 0; i < s.length(); i++) {
            char c = s.charAt(i);
            int idx = window.indexOf(String.valueOf(c));
            if (idx != -1) {
                window.delete(0, idx + 1);      // drop everything up to and including the repeat
            }
            window.append(c);
            if (window.length() > best.length()) {
                best = window.toString();
            }
        }
        return best;
    }
}
```

### Reasoning

* **Split on what is *not* a letter**: `split("[^A-Za-z]+")` uses a negated character class, so any run of non-letters becomes one delimiter. This handles apostrophes, hyphens and digits without listing them (`It's` gives `It` and `s`; `well-known` gives `well` and `known`, so `known` wins in Sample 2).
* **A leading delimiter produces a leading empty token**, and `split` drops trailing empty strings but not leading ones. The loop tolerates this because an empty token never beats `best` (length 0 is not greater than 0). A sentence with no letters (Sample 3) yields an empty array or `[""]`, and `best` stays `""`.
* **Strict `>` keeps the first of equal lengths.** In Sample 1, `quick` and `brown` are both 5 letters and `jumped` is 6, so `jumped` wins. In Sample 4 all words have length 3, so the first one, `cat`, is returned. Using `>=` would return the last one.
* **Sliding window with a `StringBuilder`**: the builder holds the current substring with no repeated characters. For each new character, if it is already in the window, delete everything up to and including its earlier occurrence, then append it. The window is always valid, and its length is compared with the best so far.
* **`window.delete(0, idx + 1)`** removes the range `[0, idx + 1)`, so the earlier copy of the character is removed too. Deleting only up to `idx` would leave the duplicate inside the window.
* **Tracing `dvdf`** (Sample 3): window `d`, `dv`; next `d` repeats at index 0, so delete `[0, 1)` leaving `v`, then append gives `vd`; then `f` gives `vdf`, which is longer than the earlier best `dv`. A common wrong solution restarts from scratch at the repeat and misses `vdf`.
* **Tie-break**: replacing `best` only when the window is **strictly longer** ensures the earliest substring wins on a tie. In `abba`, the best is `ab` (first), not `ba`.
* **`window.indexOf(String.valueOf(c))`**: `StringBuilder.indexOf` requires a `String`. The cost is `O(window length)` per character. Since the window has no duplicates, it never exceeds the alphabet size, so the overall complexity is effectively `O(n)` for ordinary text. A `HashSet` or a `Map<Character,Integer>` last-seen index gives a strictly linear version.


---

# Bonus: Predict the Output

Short snippets on the behaviours assessments like to test. Work out each answer before reading it. The output shown is what the code actually prints.

**P1.**

```java
String a = "hi";
String b = "hi";
String c = new String("hi");
System.out.println((a == b) + " " + (a == c) + " " + a.equals(c) + " " + (a == c.intern()));
```

Output: `true false true true`

**P2.**

```java
String s = "abc";
s.toUpperCase();
s.concat("d");
System.out.println(s);
```

Output: `abc`

**P3.**

```java
StringBuilder x = new StringBuilder("ab");
StringBuilder y = new StringBuilder("ab");
System.out.println(x.equals(y) + " " + x.toString().equals(y.toString()) + " " + x.compareTo(y));
```

Output: `false true 0`

**P4.**

```java
String csv = "a,b,,c,,";
System.out.println(csv.split(",").length + " " + csv.split(",", -1).length);
```

Output: `4 6`

**P5.**

```java
String j = "Java";
System.out.println(j.substring(1, 3) + "|" + j.substring(4) + "|" + j.indexOf("va") + "|" + j.indexOf('z'));
```

Output: `av||2|-1`

**P6.**

```java
String p = "a.b.c";
System.out.println(p.replace(".", "-") + " " + p.replaceAll(".", "-"));
```

Output: `a-b-c -----`

**P7.**

```java
String n = null;
System.out.println(n + "x");
```

Output: `nullx`

**P8.**

```java
System.out.println(1 + 2 + "3" + 4 + 5);
```

Output: `3345`

**P9.**

```java
StringBuilder sb = new StringBuilder("abc");
sb.insert(1, "XY").reverse().deleteCharAt(0);
System.out.println(sb);
```

Output: `bYXa`

**P10.**

```java
System.out.println("abc".compareTo("abd") + " " + "apple".compareTo("app") + " " + "A".compareTo("a"));
```

Output: `-1 2 -32`

**P11.**

```java
StringBuilder sb = new StringBuilder("abc");
System.out.println(sb.length() + " " + sb.capacity());
```

Output: `3 19`

**P12.**

```java
char c = 'a';
System.out.println((c + 1) + " " + (char) (c + 1) + " " + c + 1);
```

Output: `98 b a1`

**P13.**

```java
String t = "  Hi  ";
System.out.println("[" + t.trim() + "] [" + t.toUpperCase() + "] " + t.length());
```

Output: `[Hi] [  HI  ] 6`

**P14.**

```java
String u = "level";
System.out.println(new StringBuilder(u).reverse().toString().equals(u));
```

Output: `true`

---

# Quick Revision: Methods and Traps

## `String` methods

| Purpose | Methods | Watch out for |
|---|---|---|
| Inspect | `length()`, `charAt(i)`, `isEmpty()`, `isBlank()` (11+) | `charAt` out of range throws; `isEmpty()` is `false` for `"  "` |
| Search | `indexOf(x)`, `indexOf(x, from)`, `lastIndexOf(x)`, `contains(s)`, `startsWith(s)`, `endsWith(s)`, `matches(regex)` | `indexOf` returns `-1` when absent; `matches` must match the **whole** string; `contains` is case-sensitive |
| Extract | `substring(b)`, `substring(b, e)`, `split(regex)`, `split(regex, limit)`, `toCharArray()`, `chars()` | `e` is exclusive; `split` drops trailing empty strings (use limit `-1` to keep them); a leading empty token appears when the text starts with a delimiter |
| Compare | `equals`, `equalsIgnoreCase`, `compareTo`, `compareToIgnoreCase`, `contentEquals`, `regionMatches` | `==` compares references; `compareTo` returns any sign-carrying value, not just -1/0/1 |
| Transform | `toUpperCase`, `toLowerCase`, `trim`, `strip` (11+), `replace(a, b)`, `replaceAll(regex, r)`, `replaceFirst`, `concat`, `repeat(n)` (11+) | All return **new** strings; the original never changes. `replace` is literal, `replaceAll` is regex |
| Build / convert | `String.valueOf(x)`, `String.join(sep, parts)`, `String.format(...)`, `Integer.parseInt(s)`, `Double.parseDouble(s)` | `parseInt` throws `NumberFormatException`; `format` needs matching argument types and count |
| Pool | `intern()`, literals | Two equal literals are the same object; `new String("x")` is not |

## `StringBuilder` / `StringBuffer` methods

| Purpose | Methods | Watch out for |
|---|---|---|
| Add | `append(x)`, `insert(i, x)` | `append(char)` vs `append(int)` differ; `insert` shifts the tail right |
| Remove / edit | `delete(start, end)`, `deleteCharAt(i)`, `replace(start, end, str)`, `setCharAt(i, c)`, `setLength(n)` | `end` exclusive; `setLength` larger than the length pads with `'\u0000'` |
| Reorder | `reverse()` | Changes the builder itself and returns it |
| Read | `length()`, `capacity()`, `charAt(i)`, `indexOf(String)`, `lastIndexOf(String)`, `substring(...)`, `toString()` | `indexOf` takes a `String` (not a `char`); `substring` returns a `String`, not a builder |
| Compare | `toString().equals(...)`, `compareTo` (11+) | `equals` is **not** overridden: it compares references |

`StringBuffer` has the same API as `StringBuilder`, with every method `synchronized`. Prefer `StringBuilder` unless the object is shared between threads.

## Habits that prevent lost marks

1. Strings are **immutable**: `s.trim();` on its own line does nothing. Assign the result.
2. Use `equals`, never `==`, for content. Put the literal first when the variable may be `null`: `"ADMIN".equals(role)`.
3. Guard `charAt(0)` and `substring(...)` against empty strings, and test `indexOf` results against `-1` before using them.
4. Build strings in loops with `StringBuilder`, not with `+=`.
5. Decide **case sensitivity** for every comparison. Normalise once (`toLowerCase()`) and stay consistent.
6. `split` with a regex: `"."` and `"|"` are special, so use `"\\."` and `"\\|"`. Use `"\\s+"` for runs of whitespace and `trim()` first.
7. For a reversed string, `new StringBuilder(s).reverse().toString()`.