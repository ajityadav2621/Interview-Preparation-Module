# String Coding Questions

## 1. Reverse a String

**Problem**: Reverse a string without using built-in methods.

```java
public class ReverseString {
    // Two-pointer approach
    public static String reverse(String str) {
        if (str == null || str.length() <= 1) return str;
        char[] chars = str.toCharArray();
        int left = 0, right = chars.length - 1;
        while (left < right) {
            char temp = chars[left];
            chars[left] = chars[right];
            chars[right] = temp;
            left++;
            right--;
        }
        return new String(chars);
    }

    // Using StringBuilder
    public static String reverseWithBuilder(String str) {
        return new StringBuilder(str).reverse().toString();
    }

    public static void main(String[] args) {
        System.out.println(reverse("hello"));  // "olleh"
    }
}
```

**Time**: O(n) | **Space**: O(n) for char array

---

## 2. Check if Two Strings are Anagrams

**Problem**: Determine if two strings are anagrams (same characters, different order).

```java
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;

public class AnagramCheck {
    // Approach 1: Sort and compare
    public static boolean isAnagramSort(String s1, String s2) {
        if (s1.length() != s2.length()) return false;
        char[] c1 = s1.toCharArray();
        char[] c2 = s2.toCharArray();
        Arrays.sort(c1);
        Arrays.sort(c2);
        return Arrays.equals(c1, c2);
    }

    // Approach 2: Character count (ASCII)
    public static boolean isAnagramCount(String s1, String s2) {
        if (s1.length() != s2.length()) return false;
        int[] count = new int[256];  // ASCII
        for (int i = 0; i < s1.length(); i++) {
            count[s1.charAt(i)]++;
            count[s2.charAt(i)]--;
        }
        for (int c : count) {
            if (c != 0) return false;
        }
        return true;
    }

    // Approach 3: HashMap (Unicode)
    public static boolean isAnagramMap(String s1, String s2) {
        if (s1.length() != s2.length()) return false;
        Map<Character, Integer> map = new HashMap<>();
        for (char c : s1.toCharArray()) {
            map.put(c, map.getOrDefault(c, 0) + 1);
        }
        for (char c : s2.toCharArray()) {
            Integer count = map.get(c);
            if (count == null) return false;
            if (count == 1) map.remove(c);
            else map.put(c, count - 1);
        }
        return map.isEmpty();
    }
}
```

**Time**: O(n log n) sort, O(n) count | **Space**: O(1) or O(n)

---

## 3. First Non-Repeating Character

**Problem**: Find the first non-repeating character in a string.

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class FirstNonRepeating {
    // Using LinkedHashMap to preserve insertion order
    public static Character firstNonRepeating(String str) {
        Map<Character, Integer> countMap = new LinkedHashMap<>();
        for (char c : str.toCharArray()) {
            countMap.put(c, countMap.getOrDefault(c, 0) + 1);
        }
        for (Map.Entry<Character, Integer> entry : countMap.entrySet()) {
            if (entry.getValue() == 1) {
                return entry.getKey();
            }
        }
        return null;
    }

    // Using array (ASCII only)
    public static Character firstNonRepeatingArray(String str) {
        int[] count = new int[256];
        for (char c : str.toCharArray()) {
            count[c]++;
        }
        for (char c : str.toCharArray()) {
            if (count[c] == 1) return c;
        }
        return null;
    }

    public static void main(String[] args) {
        System.out.println(firstNonRepeating("swiss"));  // 'w'
    }
}
```

**Time**: O(n) | **Space**: O(1) for ASCII, O(n) for Unicode

---

## 4. String Permutations (Check if One is Permutation of Another)

**Problem**: Given two strings, check if one is a permutation of the other.

```java
public class StringPermutation {
    public static boolean isPermutation(String s1, String s2) {
        if (s1.length() != s2.length()) return false;
        int[] count = new int[128];  // ASCII
        for (int i = 0; i < s1.length(); i++) {
            count[s1.charAt(i)]++;
            count[s2.charAt(i)]--;
        }
        for (int c : count) {
            if (c != 0) return false;
        }
        return true;
    }
}
```

---

## 5. Remove Duplicates from String

**Problem**: Remove duplicate characters from a string.

```java
import java.util.LinkedHashSet;
import java.util.Set;

public class RemoveDuplicates {
    // Using LinkedHashSet (preserves order)
    public static String removeDuplicates(String str) {
        Set<Character> seen = new LinkedHashSet<>();
        for (char c : str.toCharArray()) {
            seen.add(c);
        }
        StringBuilder sb = new StringBuilder();
        for (char c : seen) {
            sb.append(c);
        }
        return sb.toString();
    }

    // Using boolean array (ASCII only)
    public static String removeDuplicatesArray(String str) {
        boolean[] seen = new boolean[256];
        StringBuilder sb = new StringBuilder();
        for (char c : str.toCharArray()) {
            if (!seen[c]) {
                seen[c] = true;
                sb.append(c);
            }
        }
        return sb.toString();
    }

    // In-place (if string is in a char array)
    public static String removeDuplicatesInPlace(String str) {
        char[] chars = str.toCharArray();
        int end = 0;
        for (int i = 0; i < chars.length; i++) {
            boolean found = false;
            for (int j = 0; j < end; j++) {
                if (chars[i] == chars[j]) {
                    found = true;
                    break;
                }
            }
            if (!found) {
                chars[end++] = chars[i];
            }
        }
        return new String(chars, 0, end);
    }
}
```

---

## 6. Check if String is a Rotation

**Problem**: Given two strings, check if one is a rotation of the other (e.g., "waterbottle" and "erbottlewat").

```java
public class StringRotation {
    public static boolean isRotation(String s1, String s2) {
        if (s1.length() != s2.length() || s1.length() == 0) return false;
        String combined = s1 + s1;
        return combined.contains(s2);
    }

    public static void main(String[] args) {
        System.out.println(isRotation("waterbottle", "erbottlewat"));  // true
    }
}
```

**Key Insight**: If `s2` is a rotation of `s1`, then `s2` will be a substring of `s1 + s1`.

---

## 7. Longest Common Prefix

**Problem**: Find the longest common prefix among an array of strings.

```java
public class LongestCommonPrefix {
    public static String longestCommonPrefix(String[] strs) {
        if (strs == null || strs.length == 0) return "";
        String prefix = strs[0];
        for (int i = 1; i < strs.length; i++) {
            while (strs[i].indexOf(prefix) != 0) {
                if (prefix.isEmpty()) return "";
                prefix = prefix.substring(0, prefix.length() - 1);
            }
        }
        return prefix;
    }

    // Vertical scanning
    public static String longestCommonPrefixVertical(String[] strs) {
        if (strs == null || strs.length == 0) return "";
        for (int i = 0; i < strs[0].length(); i++) {
            char c = strs[0].charAt(i);
            for (int j = 1; j < strs.length; j++) {
                if (i == strs[j].length() || strs[j].charAt(i) != c) {
                    return strs[0].substring(0, i);
                }
            }
        }
        return strs[0];
    }
}
```

---

## 8. Valid Parentheses (String-based)

**Problem**: Check if a string of parentheses is valid.

```java
import java.util.Stack;

public class ValidParentheses {
    public static boolean isValid(String s) {
        Stack<Character> stack = new Stack<>();
        for (char c : s.toCharArray()) {
            if (c == '(' || c == '{' || c == '[') {
                stack.push(c);
            } else {
                if (stack.isEmpty()) return false;
                char top = stack.pop();
                if ((c == ')' && top != '(') ||
                    (c == '}' && top != '{') ||
                    (c == ']' && top != '[')) {
                    return false;
                }
            }
        }
        return stack.isEmpty();
    }
}
```

---

## 9. Count Vowels and Consonants

```java
public class VowelConsonantCount {
    public static void count(String str) {
        int vowels = 0, consonants = 0;
        str = str.toLowerCase();
        for (char c : str.toCharArray()) {
            if (c >= 'a' && c <= 'z') {
                if (c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u') {
                    vowels++;
                } else {
                    consonants++;
                }
            }
        }
        System.out.println("Vowels: " + vowels + ", Consonants: " + consonants);
    }
}
```

---

## 10. Palindrome Check

```java
public class PalindromeCheck {
    // Simple approach
    public static boolean isPalindrome(String str) {
        str = str.replaceAll("[^a-zA-Z0-9]", "").toLowerCase();
        int left = 0, right = str.length() - 1;
        while (left < right) {
            if (str.charAt(left) != str.charAt(right)) return false;
            left++;
            right--;
        }
        return true;
    }

    // Using StringBuilder
    public static boolean isPalindromeReverse(String str) {
        str = str.replaceAll("[^a-zA-Z0-9]", "").toLowerCase();
        return str.equals(new StringBuilder(str).reverse().toString());
    }
}
```

---

## Key Patterns to Remember

| Pattern | When to Use |
|---------|-------------|
| Two pointers | Reversing, palindrome, partitioning |
| Character counting | Anagram, frequency analysis |
| Sliding window | Substring problems |
| Stack | Parentheses matching, parsing |
| StringBuilder | Repeated string building |
| `indexOf` trick | Rotation, substring search |
