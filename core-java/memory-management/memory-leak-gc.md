# How Can a Java Application Have a Memory Leak Despite Having Garbage Collection?

## The Paradox

Java's Garbage Collector (GC) automatically reclaims memory from objects that are no longer reachable. So how can a memory leak occur?

**Answer**: A memory leak in Java happens when objects are **still reachable** (referenced) but **no longer needed** by the application. The GC can't reclaim them because they're still "alive" from the GC's perspective.

## Common Causes of Java Memory Leaks

### 1. Static References (Most Common)

```java
public class StaticLeak {
    // Static reference holds objects forever
    private static List<Object> cache = new ArrayList<>();

    public static void addToCache(Object obj) {
        cache.add(obj);  // Objects are never removed!
    }
}
```

**Problem**: Static fields live as long as the JVM. If they hold references to objects, those objects can never be garbage collected.

**Fix**: Use bounded caches with eviction policies:

```java
private static final Map<String, Object> cache = 
    new LinkedHashMap<String, Object>(100, 0.75f, true) {
        protected boolean removeEldestEntry(Map.Entry<String, Object> eldest) {
            return size() > 1000;  // Max 1000 entries
        }
    };
```

### 2. Unclosed Resources

```java
// Leaking file handles
public void readFile(String filename) throws IOException {
    FileInputStream fis = new FileInputStream(filename);
    // Process file...
    // Missing: fis.close() — file handle leaks!
}
```

**Fix**: Use try-with-resources:

```java
public void readFile(String filename) throws IOException {
    try (FileInputStream fis = new FileInputStream(filename)) {
        // Process file...
    }  // Automatically closed
}
```

### 3. Inner Classes Holding Outer Class References

```java
public class Outer {
    private byte[] largeData = new byte[1024 * 1024];  // 1MB

    public class Inner {
        public void doSomething() {
            // Inner class holds implicit reference to Outer
        }
    }

    public Inner createInner() {
        return new Inner();
    }
}

// Problem:
Outer outer = new Outer();
Outer.Inner inner = outer.createInner();
outer = null;  // Outer is NOT garbage collected!
               // Inner still holds a reference to it
```

**Fix**: Use static nested classes when possible:

```java
public static class StaticNested {
    // No implicit reference to Outer
}
```

### 4. ThreadLocal Leaks

```java
public class ThreadLocalLeak {
    private static ThreadLocal<SimpleDateFormat> formatter = 
        new ThreadLocal<SimpleDateFormat>() {
            @Override
            protected SimpleDateFormat initialValue() {
                return new SimpleDateFormat("yyyy-MM-dd");
            }
        };
}
```

**Problem**: In application servers (Tomcat, Jetty), `ThreadLocal` values are stored in the thread's `ThreadLocalMap`. When the application is undeployed, the threads (from the thread pool) persist, and the `ThreadLocal` values are not cleaned up — the entire web application class loader is leaked.

**Fix**: Always remove `ThreadLocal` values:

```java
try {
    formatter.get().format(date);
} finally {
    formatter.remove();  // Clean up
}
```

### 5. Listeners and Callbacks

```java
public class EventSource {
    private List<EventListener> listeners = new ArrayList<>();

    public void addListener(EventListener listener) {
        listeners.add(listener);
    }
    // Missing: removeListener() method!
}
```

**Problem**: If listeners are added but never removed, they (and their referenced objects) are never garbage collected.

**Fix**: Provide a `removeListener()` method and ensure callers use it.

### 6. Anonymous Classes Capturing Variables

```java
public class CapturingLeak {
    private byte[] largeData = new byte[1024 * 1024];

    public Runnable createTask() {
        // Anonymous class captures 'this' — holds reference to largeData
        return new Runnable() {
            public void run() {
                System.out.println("Task running");
            }
        };
    }
}
```

**Fix**: Use static nested classes or extract the task to a separate class.

### 7. Caches Without Eviction

```java
public class CacheLeak {
    private Map<String, Object> cache = new HashMap<>();

    public Object get(String key) {
        Object value = cache.get(key);
        if (value == null) {
            value = loadFromDatabase(key);
            cache.put(key, value);  // Never evicted!
        }
        return value;
    }
}
```

**Fix**: Use `WeakHashMap` or a proper caching library:

```java
// WeakHashMap — entries are removed when keys are no longer referenced
private Map<String, Object> cache = new WeakHashMap<>();

// Or use a proper cache with size/time-based eviction
private Cache<String, Object> cache = Caffeine.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build();
```

### 8. String Intern Leaks

```java
// In older Java versions, String.intern() could cause PermGen leaks
for (int i = 0; i < 1000000; i++) {
    String interned = new String("str" + i).intern();
    // Each interned string stays in PermGen forever
}
```

**Fix**: Be cautious with `String.intern()` in loops.

## How to Detect Memory Leaks

### 1. Monitor Heap Usage

```bash
# Use jstat to monitor GC
jstat -gc <pid> 1s

# Use jmap to take heap dumps
jmap -dump:format=b,file=heap.hprof <pid>
```

### 2. Analyze Heap Dumps

Use tools like:
- **Eclipse MAT** (Memory Analyzer Tool)
- **VisualVM**
- **JProfiler**

### 3. Monitor with JMX

```java
MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
System.out.println("Used: " + heapUsage.getUsed() / 1024 / 1024 + " MB");
```

### 4. Add Memory Leak Detection

```java
// Monitor for suspiciously growing collections
public class LeakDetector {
    private final Map<String, Integer> collectionSizes = new ConcurrentHashMap<>();

    public void monitorCollection(String name, int size) {
        Integer previous = collectionSizes.get(name);
        if (previous != null && size > previous * 2) {
            System.err.println("WARNING: " + name + " grew from " + 
                previous + " to " + size);
        }
        collectionSizes.put(name, size);
    }
}
```

## Prevention Strategies

1. **Use weak references** for caches: `WeakHashMap`, `WeakReference`
2. **Always close resources** — use try-with-resources
3. **Remove listeners/callbacks** when no longer needed
4. **Avoid static collections** that grow unbounded
5. **Clean up ThreadLocal** values in `finally` blocks
6. **Use proper caching libraries** (Caffeine, Ehcache) with eviction policies
7. **Profile regularly** — use heap dumps and memory analyzers
8. **Set appropriate JVM flags** for GC tuning

## Key Takeaway

> Java memory leaks occur when objects are still reachable but no longer needed. The GC can only reclaim unreachable objects. Common causes include static references, unclosed resources, ThreadLocal leaks, and caches without eviction. Prevention requires careful resource management, proper use of weak references, and regular profiling.
