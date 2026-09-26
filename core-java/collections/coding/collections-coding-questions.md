# Collections Coding Questions

## 1. Implement a HashMap from Scratch

```java
public class MyHashMap<K, V> {
    private static final int DEFAULT_CAPACITY = 16;
    private static final float DEFAULT_LOAD_FACTOR = 0.75f;

    private Entry<K, V>[] table;
    private int size;
    private final float loadFactor;
    private int threshold;

    static class Entry<K, V> {
        final int hash;
        final K key;
        V value;
        Entry<K, V> next;

        Entry(int hash, K key, V value, Entry<K, V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }

    public MyHashMap() {
        this(DEFAULT_CAPACITY, DEFAULT_LOAD_FACTOR);
    }

    public MyHashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    public MyHashMap(int initialCapacity, float loadFactor) {
        this.loadFactor = loadFactor;
        this.threshold = (int) (initialCapacity * loadFactor);
        this.table = new Entry[initialCapacity];
    }

    private int hash(K key) {
        if (key == null) return 0;
        int h;
        return (h = key.hashCode()) ^ (h >>> 16);
    }

    private int indexFor(int hash, int length) {
        return (length - 1) & hash;
    }

    public void put(K key, V value) {
        if (size >= threshold) resize();
        int hash = hash(key);
        int index = indexFor(hash, table.length);

        for (Entry<K, V> e = table[index]; e != null; e = e.next) {
            if (e.hash == hash && Objects.equals(e.key, key)) {
                e.value = value;
                return;
            }
        }
        table[index] = new Entry<>(hash, key, value, table[index]);
        size++;
    }

    public V get(K key) {
        int hash = hash(key);
        int index = indexFor(hash, table.length);
        for (Entry<K, V> e = table[index]; e != null; e = e.next) {
            if (e.hash == hash && Objects.equals(e.key, key)) {
                return e.value;
            }
        }
        return null;
    }

    public V remove(K key) {
        int hash = hash(key);
        int index = indexFor(hash, table.length);
        Entry<K, V> prev = null;
        for (Entry<K, V> e = table[index]; e != null; e = e.next) {
            if (e.hash == hash && Objects.equals(e.key, key)) {
                if (prev == null) {
                    table[index] = e.next;
                } else {
                    prev.next = e.next;
                }
                size--;
                return e.value;
            }
            prev = e;
        }
        return null;
    }

    private void resize() {
        Entry<K, V>[] oldTable = table;
        int oldCapacity = oldTable.length;
        int newCapacity = oldCapacity * 2;
        Entry<K, V>[] newTable = new Entry[newCapacity];

        for (int i = 0; i < oldCapacity; i++) {
            for (Entry<K, V> e = oldTable[i]; e != null; e = e.next) {
                int index = indexFor(e.hash, newCapacity);
                e.next = newTable[index];
                newTable[index] = e;
            }
        }

        table = newTable;
        threshold = (int) (newCapacity * loadFactor);
    }

    public int size() {
        return size;
    }
}
```

---

## 2. LRU Cache (Using LinkedHashMap)

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);  // accessOrder = true
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }

    public static void main(String[] args) {
        LRUCache<Integer, String> cache = new LRUCache<>(2);
        cache.put(1, "one");
        cache.put(2, "two");
        cache.get(1);  // Access key 1 — makes it most recently used
        cache.put(3, "three");  // Evicts key 2 (LRU)
        System.out.println(cache.get(2));  // null — was evicted
    }
}
```

---

## 3. Find First Non-Repeating Character

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class FirstNonRepeating {
    public static char firstNonRepeating(String str) {
        Map<Character, Integer> countMap = new LinkedHashMap<>();
        for (char c : str.toCharArray()) {
            countMap.put(c, countMap.getOrDefault(c, 0) + 1);
        }
        for (Map.Entry<Character, Integer> entry : countMap.entrySet()) {
            if (entry.getValue() == 1) {
                return entry.getKey();
            }
        }
        return '\0';  // or throw exception
    }
}
```

---

## 4. Group Anagrams

```java
import java.util.*;

public class GroupAnagrams {
    public static List<List<String>> groupAnagrams(String[] strs) {
        Map<String, List<String>> map = new HashMap<>();
        for (String str : strs) {
            char[] chars = str.toCharArray();
            Arrays.sort(chars);
            String key = new String(chars);
            map.computeIfAbsent(key, k -> new ArrayList<>()).add(str);
        }
        return new ArrayList<>(map.values());
    }
}
```

---

## 5. Two Sum

```java
import java.util.*;

public class TwoSum {
    public static int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> map = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            int complement = target - nums[i];
            if (map.containsKey(complement)) {
                return new int[]{map.get(complement), i};
            }
            map.put(nums[i], i);
        }
        return new int[0];
    }
}
```

---

## 6. Remove Duplicates from Sorted Array (InPlace)

```java
public class RemoveDuplicates {
    public static int removeDuplicates(int[] nums) {
        if (nums.length == 0) return 0;
        int uniqueIndex = 1;
        for (int i = 1; i < nums.length; i++) {
            if (nums[i] != nums[i - 1]) {
                nums[uniqueIndex++] = nums[i];
            }
        }
        return uniqueIndex;
    }
}
```

---

## 7. Merge Intervals

```java
import java.util.*;

public class MergeIntervals {
    public static int[][] merge(int[][] intervals) {
        if (intervals.length <= 1) return intervals;

        Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));

        List<int[]> merged = new ArrayList<>();
        int[] current = intervals[0];

        for (int i = 1; i < intervals.length; i++) {
            if (current[1] >= intervals[i][0]) {
                current[1] = Math.max(current[1], intervals[i][1]);
            } else {
                merged.add(current);
                current = intervals[i];
            }
        }
        merged.add(current);

        return merged.toArray(new int[0][]);
    }
}
```

---

## 8. Queue Using Two Stacks

```java
import java.util.Stack;

public class MyQueue<T> {
    private Stack<T> inStack = new Stack<>();
    private Stack<T> outStack = new Stack<>();

    public void offer(T value) {
        inStack.push(value);
    }

    public T poll() {
        if (outStack.isEmpty()) {
            while (!inStack.isEmpty()) {
                outStack.push(inStack.pop());
            }
        }
        if (outStack.isEmpty()) throw new java.util.NoSuchElementException();
        return outStack.pop();
    }

    public T peek() {
        if (outStack.isEmpty()) {
            while (!inStack.isEmpty()) {
                outStack.push(inStack.pop());
            }
        }
        if (outStack.isEmpty()) throw new java.util.NoSuchElementException();
        return outStack.peek();
    }

    public boolean isEmpty() {
        return inStack.isEmpty() && outStack.isEmpty();
    }
}
```

---

## 9. Min Stack (O(1) getMin)

```java
import java.util.Stack;

public class MinStack {
    private Stack<Integer> stack = new Stack<>();
    private Stack<Integer> minStack = new Stack<>();

    public void push(int val) {
        stack.push(val);
        if (minStack.isEmpty() || val <= minStack.peek()) {
            minStack.push(val);
        }
    }

    public int pop() {
        if (stack.isEmpty()) throw new java.util.EmptyStackException();
        int val = stack.pop();
        if (val == minStack.peek()) {
            minStack.pop();
        }
        return val;
    }

    public int top() {
        return stack.peek();
    }

    public int getMin() {
        return minStack.peek();
    }
}
```

---

## 10. Valid Parentheses

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

## Key Patterns

| Pattern | Data Structure | Use Case |
|---------|---------------|----------|
| Hash map for lookup | HashMap | Two Sum, Anagram grouping |
| Two pointers | Array/List | Remove duplicates, merge intervals |
| Stack for matching | Stack | Valid parentheses, min stack |
| Queue via stacks | Two Stacks | Queue implementation |
| LinkedHashMap for LRU | LinkedHashMap | Cache with eviction policy |
| Sorting + greedy | Array | Merge intervals, meeting rooms |
