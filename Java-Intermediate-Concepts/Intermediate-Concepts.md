# Java Intermediate Concepts - Collections, Exceptions, and Threads

## 1. Collections Framework

### List Interface Implementations
```java
// ArrayList - Dynamic array
List<String> list1 = new ArrayList<>();
// Fast random access (O(1)), slow insertion/deletion in middle (O(n))

// LinkedList - Doubly linked list
List<String> list2 = new LinkedList<>();
// Slow random access (O(n)), fast insertion/deletion in middle (O(1))

// Vector - Thread-safe ArrayList (legacy)
List<String> vector = new Vector<>();
```

### Set Interface
```java
// HashSet - Uses HashMap, no order, O(1) operations
Set<String> set1 = new HashSet<>();

// TreeSet - Uses TreeMap, sorted order, O(log n) operations
Set<String> set2 = new TreeSet<>();

// LinkedHashSet - HashSet with insertion order
Set<String> set3 = new LinkedHashSet<>();
```

### Map Interface
```java
// HashMap - Hash table, null key/value allowed
Map<String, Integer> map1 = new HashMap<>();

// TreeMap - Red-black tree, sorted by keys
Map<String, Integer> map2 = new TreeMap<>();

// Hashtable - Thread-safe HashMap (legacy, no null)
Map<String, Integer> hashtable = new Hashtable<>();
```

## 2. Exception Handling

### Exception Hierarchy
```
Throwable
├── Error (JVM errors, shouldn't catch)
└── Exception
    ├── IOException (File operations)
    ├── RuntimeException (Unchecked)
    └── Others (Checked exceptions)
```

### Custom Exception
```java
class CustomException extends Exception {
    public CustomException(String message) {
        super(message);
    }
}

// Usage
public void process() throws CustomException {
    if (someCondition) {
        throw new CustomException("Error occurred");
    }
}
```

### Try-Catch-Finally
```java
try {
    // Code that might throw
} catch (SpecificException e) {
    // Handle specific exception
} catch (AnotherException e) {
    // Handle another exception
} finally {
    // Always executes (cleanup)
}
```

## 3. Multithreading Basics

### Thread Creation
```java
// Method 1: Extend Thread class
class MyThread extends Thread {
    public void run() {
        System.out.println("Thread running");
    }
}

// Method 2: Implement Runnable
class MyRunnable implements Runnable {
    public void run() {
        System.out.println("Runnable running");
    }
}

// Usage
new MyThread().start();
new Thread(new MyRunnable()).start();
```

### Thread States
- NEW → RUNNABLE → BLOCKED/WAITING → TERMINATED
- TIMED_WAITING (sleep, wait with timeout)

## 4. Synchronization

### synchronized Methods
```java
class Counter {
    private int count = 0;
    
    public synchronized void increment() {
        count++; // Thread-safe
    }
    
    public synchronized int getCount() {
        return count;
    }
}
```

### synchronized Blocks
```java
class SharedResource {
    private Object lock = new Object();
    private int value;
    
    public void setValue(int value) {
        synchronized(lock) { // Lock on specific object
            this.value = value;
        }
    }
}
```

## 5. Inner Classes

### Types of Inner Classes
```java
// Member Inner Class
class Outer {
    int x = 10;
    
    class Inner {
        int y = 20;
        
        void display() {
            System.out.println(x + y); // Can access outer variables
        }
    }
    
    // Static Inner Class
    static class StaticInner {
        static int z = 30;
        // Cannot access non-static outer variables
    }
    
    // Local Inner Class (inside method)
    void method() {
        class LocalInner {
            void display() {
                System.out.println("Local inner class");
            }
        }
    }
    
    // Anonymous Inner Class
    Runnable r = new Runnable() {
        public void run() {
            System.out.println("Anonymous class");
        }
    };
}
```

## Intermediate Coding Questions

### Q1: Two Sum
```java
public int[] twoSum(int[] nums, int target) {
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
```

### Q2: Valid Parentheses
```java
public boolean isValid(String s) {
    Stack<Character> stack = new Stack<>();
    for (char c : s.toCharArray()) {
        if (c == '(' || c == '[' || c == '{') {
            stack.push(c);
        } else {
            if (stack.isEmpty()) return false;
            char top = stack.pop();
            if ((c == ')' && top != '(') ||
                (c == ']' && top != '[') ||
                (c == '}' && top != '{')) {
                return false;
            }
        }
    }
    return stack.isEmpty();
}
```

### Q3: Merge Two Sorted Lists
```java
class ListNode {
    int val;
    ListNode next;
    ListNode() {}
    ListNode(int val) { this.val = val; }
    ListNode(int val, ListNode next) { this.val = val; this.next = next; }
}

public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0);
    ListNode current = dummy;
    
    while (l1 != null && l2 != null) {
        if (l1.val <= l2.val) {
            current.next = l1;
            l1 = l1.next;
        } else {
            current.next = l2;
            l2 = l2.next;
        }
        current = current.next;
    }
    
    current.next = (l1 != null) ? l1 : l2;
    return dummy.next;
}
```

### Q4: Maximum Subarray Sum (Kadane's Algorithm)
```java
public int maxSubArray(int[] nums) {
    int maxSoFar = nums[0];
    int maxEndingHere = nums[0];
    
    for (int i = 1; i < nums.length; i++) {
        maxEndingHere = Math.max(nums[i], maxEndingHere + nums[i]);
        maxSoFar = Math.max(maxSoFar, maxEndingHere);
    }
    
    return maxSoFar;
}
```

### Q5: Binary Tree Inorder Traversal
```java
class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;
    TreeNode() {}
    TreeNode(int val) { this.val = val; }
}

public List<Integer> inorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    inorderHelper(root, result);
    return result;
}

private void inorderHelper(TreeNode node, List<Integer> result) {
    if (node != null) {
        inorderHelper(node.left, result);
        result.add(node.val);
        inorderHelper(node.right, result);
    }
}
```