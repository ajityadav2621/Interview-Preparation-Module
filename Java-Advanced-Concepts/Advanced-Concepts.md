# Java Advanced Concepts - Concurrency, Memory, and Functional Programming

## 1. Java Memory Model (JMM)

### Key Concepts
- **Visibility**: Changes made by one thread visible to others
- **Atomicity**: Operations complete without interruption
- **Ordering**: Execution order guarantees

### Volatile Keyword
```java
class SharedObject {
    volatile int counter; // Ensures visibility across threads
    
    void increment() {
        counter++; // Not atomic! Need synchronization
    }
}
```

## 2. Concurrent Collections

### ConcurrentHashMap
```java
Map<String, Integer> map = new ConcurrentHashMap<>();
// Thread-safe operations
map.putIfAbsent("key", 1);
map.computeIfAbsent("key", k -> 1);
```

### CopyOnWriteArrayList
```java
List<String> list = new CopyOnWriteArrayList<>();
// Thread-safe for read-heavy workloads
// Creates new copy on modification
```

## 3. Atomic Variables

### AtomicInteger
```java
AtomicInteger atomicCounter = new AtomicInteger(0);

atomicCounter.incrementAndGet(); // Thread-safe increment
atomicCounter.compareAndSet(0, 1); // Atomic CAS operation
```

## 4. Lock Framework

### ReentrantLock
```java
ReentrantLock lock = new ReentrantLock(true); // Fair lock

lock.lock();
try {
    // Critical section
} finally {
    lock.unlock(); // Must unlock
}

// Try lock with timeout
if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
    try {
        // Critical section
    } finally {
        lock.unlock();
    }
}
```

## 5. Thread Pools

### Executor Framework
```java
// Fixed thread pool
ExecutorService fixedPool = Executors.newFixedThreadPool(10);

// Cached thread pool
ExecutorService cachedPool = Executors.newCachedThreadPool();

// Scheduled thread pool
ScheduledExecutorService scheduledPool = Executors.newScheduledThreadPool(5);

// Submit tasks
Future<String> future = fixedPool.submit(() -> {
    return "Result";
});

// Get result
String result = future.get(); // Blocks until complete
```

## 6. Functional Programming

### Streams API
```java
List<String> names = Arrays.asList("John", "Jane", "Alice");

// Filter and map
List<String> filtered = names.stream()
    .filter(name -> name.startsWith("J"))
    .map(String::toUpperCase)
    .collect(Collectors.toList());

// Parallel processing
List<String> parallel = names.parallelStream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

### Optional
```java
Optional<String> optional = Optional.of("Hello");

optional.ifPresent(System.out::println);
optional.orElse("Default");
optional.orElseThrow(() -> new RuntimeException("Not found"));
```

## Advanced Coding Questions

### Q1: LRU Cache
```java
class LRUCache {
    private final int capacity;
    private final LinkedHashMap<Integer, Integer> map;
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new LinkedHashMap<Integer, Integer>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
                return size() > capacity;
            }
        };
    }
    
    public int get(int key) {
        return map.getOrDefault(key, -1);
    }
    
    public void put(int key, int value) {
        map.put(key, value);
    }
}
```

### Q2: Binary Tree Maximum Path Sum
```java
class Solution {
    private int maxSum = Integer.MIN_VALUE;
    
    public int maxPathSum(TreeNode root) {
        maxPathHelper(root);
        return maxSum;
    }
    
    private int maxPathHelper(TreeNode node) {
        if (node == null) return 0;
        
        int left = Math.max(0, maxPathHelper(node.left));
        int right = Math.max(0, maxPathHelper(node.right));
        
        maxSum = Math.max(maxSum, left + right + node.val);
        
        return Math.max(left, right) + node.val;
    }
}
```

### Q3: Word Ladder
```java
public int ladderLength(String beginWord, String endWord, List<String> wordList) {
    Set<String> wordSet = new HashSet<>(wordList);
    if (!wordSet.contains(endWord)) return 0;
    
    Queue<String> queue = new LinkedList<>();
    queue.offer(beginWord);
    int level = 1;
    
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            String word = queue.poll();
            
            if (word.equals(endWord)) return level;
            
            char[] chars = word.toCharArray();
            for (int j = 0; j < chars.length; j++) {
                char original = chars[j];
                for (char c = 'a'; c <= 'z'; c++) {
                    if (c == original) continue;
                    chars[j] = c;
                    String newWord = new String(chars);
                    if (wordSet.contains(newWord)) {
                        queue.offer(newWord);
                        wordSet.remove(newWord);
                    }
                }
                chars[j] = original;
            }
        }
        level++;
    }
    return 0;
}
```

### Q4: Median of Data Stream
```java
class MedianFinder {
    private PriorityQueue<Integer> maxHeap; // smaller half
    private PriorityQueue<Integer> minHeap; // larger half
    
    public MedianFinder() {
        maxHeap = new PriorityQueue<>(Collections.reverseOrder());
        minHeap = new PriorityQueue<>();
    }
    
    public void addNum(int num) {
        if (maxHeap.isEmpty() || num <= maxHeap.peek()) {
            maxHeap.offer(num);
        } else {
            minHeap.offer(num);
        }
        
        // Balance heaps
        if (maxHeap.size() > minHeap.size() + 1) {
            minHeap.offer(maxHeap.poll());
        } else if (minHeap.size() > maxHeap.size()) {
            maxHeap.offer(minHeap.poll());
        }
    }
    
    public double findMedian() {
        if (maxHeap.size() == minHeap.size()) {
            return (maxHeap.peek() + minHeap.peek()) / 2.0;
        }
        return maxHeap.peek();
    }
}
```

### Q5: Longest Consecutive Sequence
```java
public int longestConsecutive(int[] nums) {
    Set<Integer> set = new HashSet<>();
    for (int num : nums) set.add(num);
    
    int longest = 0;
    for (int num : set) {
        if (!set.contains(num - 1)) {
            int current = num;
            int streak = 1;
            
            while (set.contains(current + 1)) {
                current++;
                streak++;
            }
            
            longest = Math.max(longest, streak);
        }
    }
    return longest;
}
```