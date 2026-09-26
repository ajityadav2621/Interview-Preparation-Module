# Java Memory Leaks Despite Garbage Collection

## What is a Memory Leak?
Memory leak occurs when objects are no longer needed but cannot be garbage collected because they're still referenced.

## Common Causes

### 1. **Static Collections**
```java
// BAD - grows indefinitely
private static List<String> cache = new ArrayList<>();

public void processData(String data) {
    cache.add(data); // Never cleared
}
```

### 2. **ThreadLocal Misuse**
```java
// BAD - ThreadLocal retains values per thread
private static ThreadLocal<User> currentUser = new ThreadLocal<>();

// In servlet or thread pool, values accumulate
public void processRequest() {
    currentUser.set(user); // Never cleared
}
```

### 3. **Unbounded Caches**
```java
// BAD - no eviction policy
private static Map<String, Object> cache = new HashMap<>();

public Object get(String key) {
    if (!cache.containsKey(key)) {
        cache.put(key, expensiveOperation());
    }
    return cache.get(key);
}
```

### 4. **Listeners and Callbacks**
```java
// BAD - listener never removed
public class Button {
    private List<EventListener> listeners = new ArrayList<>();
    
    public void addListener(EventListener listener) {
        listeners.add(listener);
    }
    // No removeListener method!
}
```

## How to Identify

### Tools
- **jstat**: Monitor GC activity
- **jmap**: Get heap dump
- **jhat/VisualVM**: Analyze heap dump
- **Flight Recorder**: Continuous profiling

### Signs
- Increasing memory usage over time
- Frequent Full GC
- OutOfMemoryError
- High heap usage after GC

## Prevention
- Use weak references for caches
- Implement proper cleanup
- Use bounded collections
- Monitor memory usage

## Interview Tip
Explain that GC only collects unreachable objects. If objects remain referenced, they cannot be collected, leading to memory leaks.