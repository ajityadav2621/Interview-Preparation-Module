# Memory Management Coding Questions

## 1. Detect Memory Leak in a Cache

```java
// Problem: A cache that grows unbounded
public class LeakyCache {
    private static final Map<String, Object> cache = new HashMap<>();

    public static void put(String key, Object value) {
        cache.put(key, value);  // Never removes entries!
    }

    public static Object get(String key) {
        return cache.get(key);
    }
}

// Solution: Bounded cache with LRU eviction
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);  // accessOrder = true for LRU
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

---

## 2. Implement a Memory-Efficient Object Pool

```java
import java.util.Queue;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.function.Supplier;

public class ObjectPool<T> {
    private final Queue<T> pool = new ConcurrentLinkedQueue<>();
    private final Supplier<T> factory;
    private final int maxSize;

    public ObjectPool(Supplier<T> factory, int maxSize) {
        this.factory = factory;
        this.maxSize = maxSize;
    }

    public T borrow() {
        T obj = pool.poll();
        return obj != null ? obj : factory.get();
    }

    public void release(T obj) {
        if (pool.size() < maxSize) {
            pool.offer(obj);
        }
        // If pool is full, let the object be garbage collected
    }

    public int available() {
        return pool.size();
    }
}

// Usage:
ObjectPool<StringBuilder> pool = new ObjectPool<>(() -> new StringBuilder(), 10);
StringBuilder sb = pool.borrow();
sb.append("Hello");
pool.release(sb);
```

---

## 3. WeakReference Cache

```java
import java.lang.ref.WeakReference;
import java.util.concurrent.ConcurrentHashMap;

public class WeakCache<K, V> {
    private final ConcurrentHashMap<K, WeakReference<V>> cache = new ConcurrentHashMap<>();

    public void put(K key, V value) {
        cache.put(key, new WeakReference<>(value));
    }

    public V get(K key) {
        WeakReference<V> ref = cache.get(key);
        if (ref == null) return null;
        
        V value = ref.get();
        if (value == null) {
            // Value was garbage collected — clean up
            cache.remove(key);
        }
        return value;
    }

    // Clean up stale entries
    public void cleanup() {
        cache.entrySet().removeIf(entry -> entry.getValue().get() == null);
    }
}
```

---

## 4. Monitor GC and Memory Usage

```java
import java.lang.management.*;
import java.util.List;

public class GCMonitor {
    private final List<GarbageCollectorMXBean> gcBeans;
    private final MemoryMXBean memoryBean;

    public GCMonitor() {
        this.gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        this.memoryBean = ManagementFactory.getMemoryMXBean();
    }

    public void printStats() {
        System.out.println("=== GC Stats ===");
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            System.out.printf("%s: %d collections, %dms total%n",
                gcBean.getName(),
                gcBean.getCollectionCount(),
                gcBean.getCollectionTime());
        }

        System.out.println("\n=== Memory Stats ===");
        MemoryUsage heap = memoryBean.getHeapMemoryUsage();
        System.out.printf("Heap: %d MB used / %d MB max%n",
            heap.getUsed() / 1024 / 1024,
            heap.getMax() / 1024 / 1024);
    }

    public static void main(String[] args) {
        GCMonitor monitor = new GCMonitor();
        monitor.printStats();
    }
}
```

---

## 5. Implement a Memory-Efficient String Joiner

```java
// BAD: Creates many intermediate String objects
public static String joinBad(String[] parts, String delimiter) {
    String result = "";
    for (int i = 0; i < parts.length; i++) {
        result += parts[i];
        if (i < parts.length - 1) {
            result += delimiter;
        }
    }
    return result;
}

// GOOD: Uses StringBuilder
public static String joinGood(String[] parts, String delimiter) {
    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < parts.length; i++) {
        sb.append(parts[i]);
        if (i < parts.length - 1) {
            sb.append(delimiter);
        }
    }
    return sb.toString();
}

// BEST: Pre-size StringBuilder
public static String joinBest(String[] parts, String delimiter) {
    int totalLength = 0;
    for (String part : parts) {
        totalLength += part.length() + delimiter.length();
    }
    StringBuilder sb = new StringBuilder(totalLength);
    for (int i = 0; i < parts.length; i++) {
        sb.append(parts[i]);
        if (i < parts.length - 1) {
            sb.append(delimiter);
        }
    }
    return sb.toString();
}
```

---

## 6. Detect Circular References

```java
import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;
import java.lang.ref.WeakReference;

public class ReferenceDemo {
    public static void demonstrateReferences() throws InterruptedException {
        // Strong reference — prevents GC
        Object strongRef = new Object();

        // Weak reference — GC can collect the object
        WeakReference<Object> weakRef = new WeakReference<>(strongRef);

        // Phantom reference — for cleanup notification
        ReferenceQueue<Object> queue = new ReferenceQueue<>();
        PhantomReference<Object> phantomRef = new PhantomReference<>(strongRef, queue);

        // Remove strong reference
        strongRef = null;

        // Object is now eligible for GC
        System.gc();
        Thread.sleep(100);

        System.out.println("Weak ref: " + weakRef.get());  // null — GC collected it
        System.out.println("Phantom ref: " + phantomRef.get());  // null — always
        System.out.println("Phantom in queue: " + (queue.poll() != null));  // true
    }
}
```

---

## 7. Memory-Efficient List Building

```java
// BAD: Creates many intermediate arrays
public static List<Integer> filterBad(List<Integer> input, int threshold) {
    List<Integer> result = new ArrayList<>();
    for (Integer n : input) {
        if (n > threshold) {
            result.add(n);
        }
    }
    return result;
}

// GOOD: Pre-size the list
public static List<Integer> filterGood(List<Integer> input, int threshold) {
    // Estimate size to avoid resizing
    List<Integer> result = new ArrayList<>(input.size() / 2);
    for (Integer n : input) {
        if (n > threshold) {
            result.add(n);
        }
    }
    return result;
}

// BEST: Use streams with proper sizing
public static List<Integer> filterBest(List<Integer> input, int threshold) {
    return input.stream()
        .filter(n -> n > threshold)
        .collect(Collectors.toList());
}
```

---

## 8. Implement a Simple Memory Leak Detector

```java
import java.util.*;

public class MemoryLeakDetector {
    private final Map<String, Long> allocationPoints = new HashMap<>();
    private final Map<String, Integer> allocationCounts = new HashMap<>();

    public void recordAllocation(String allocationPoint) {
        allocationPoints.merge(allocationPoint, 1L, Long::sum);
        allocationCounts.merge(allocationPoint, 1, Integer::sum);
    }

    public void printReport() {
        System.out.println("=== Memory Allocation Report ===");
        allocationPoints.entrySet().stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .limit(10)
            .forEach(entry -> {
                String point = entry.getKey();
                long count = entry.getValue();
                int instances = allocationCounts.get(point);
                System.out.printf("%s: %d instances%n", point, instances);
            });
    }

    // Usage with a custom allocator
    public static void main(String[] args) {
        MemoryLeakDetector detector = new MemoryLeakDetector();
        
        // Simulate allocations
        for (int i = 0; i < 1000; i++) {
            detector.recordAllocation("MyClass.method1");
            new byte[1024];  // Allocate 1KB
        }
        
        detector.printReport();
    }
}
```

---

## Key Patterns

| Pattern | Purpose |
|---------|---------|
| Bounded collections | Prevent unbounded growth |
| Weak/Soft references | Allow GC of cached objects |
| Object pooling | Reuse expensive objects |
| Pre-sizing collections | Avoid resizing overhead |
| StringBuilder | Efficient string concatenation |
| try-with-resources | Ensure resource cleanup |
| Reference queues | Track GC of objects |
