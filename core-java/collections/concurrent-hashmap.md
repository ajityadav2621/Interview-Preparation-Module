# HashMap vs ConcurrentHashMap — What's the Real Difference?

## The Core Problem

`HashMap` is **not thread-safe**. If multiple threads access it concurrently and at least one thread modifies the map structurally, it can lead to:

1. **Infinite loops** (in Java 7, during resizing)
2. **Lost updates** (one thread's write overwrites another's)
3. **Corrupted data structures** (broken linked lists in buckets)
4. **Inconsistent reads** (seeing partial state)

## HashMap — Not Thread-Safe

```java
// DANGEROUS: HashMap is not thread-safe
Map<String, Integer> map = new HashMap<>();

// Thread 1
map.put("key1", 1);

// Thread 2 (concurrent)
map.put("key2", 2);

// Result: Potential data corruption, lost updates, or infinite loops
```

### Java 7: Infinite Loop During Resize

In Java 7, when a HashMap resizes, it reverses the linked list in each bucket. If two threads trigger resize simultaneously, they can create a **circular linked list**, causing `get()` to loop forever:

```
// Before resize:
bucket[3] -> A -> B -> C -> null

// During concurrent resize, two threads reverse the list:
// Thread 1: C -> B -> A -> null
// Thread 2: A -> B -> C -> A -> B -> C -> ... (infinite loop!)
```

### Java 8: No Infinite Loop, But Still Unsafe

Java 8 fixed the infinite loop issue, but `HashMap` is still **not thread-safe** — you can still get:
- Lost updates
- Inconsistent reads
- `NullPointerException` in certain scenarios

## Solutions for Thread-Safe Maps

### Option 1: `Collections.synchronizedMap()`

```java
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>());

// All operations are synchronized on a single lock
// Problem: Only one thread can access the map at a time — poor concurrency
```

**Issues**:
- **Coarse-grained locking**: One lock for the entire map
- **Poor concurrency**: Even read operations block each other
- **Iterator must be manually synchronized**:

```java
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
synchronized (syncMap) {
    for (Map.Entry<String, Integer> entry : syncMap.entrySet()) {
        // Must synchronize iteration manually
    }
}
```

### Option 2: `ConcurrentHashMap` (Java 5+)

```java
Map<String, Integer> map = new ConcurrentHashMap<>();

// Thread-safe with high concurrency
// Multiple threads can read simultaneously
// Writes are fine-grained (segment-level or bin-level locking)
```

## ConcurrentHashMap Internals

### Java 7: Segment-Based Locking

```
ConcurrentHashMap structure (Java 7):
┌─────────────────────────────────────────────┐
│  Segment[0]  Segment[1]  Segment[2] ...     │
│  ┌───────┐   ┌───────┐   ┌───────┐         │
│  │ Lock  │   │ Lock  │   │ Lock  │         │
│  │  +    │   │  +    │   │  +    │         │
│  │ HashE │   │ HashE │   │ HashE │         │
│  │  table│   │  table│   │  table│         │
│  └───────┘   └───────┘   └───────┘         │
└─────────────────────────────────────────────┘
```

- The map is divided into **16 segments** (by default)
- Each segment has its own **lock**
- Threads accessing different segments don't block each other
- **Reads are lock-free** (using volatile reads)
- **Writes lock only the affected segment**

### Java 8: Bin-Based Locking (No Segments)

Java 8 removed the `Segment` class entirely and uses a simpler approach:

- Uses the same `Node` structure as `HashMap`
- Uses **synchronized** on individual bins (buckets) when needed
- Uses **CAS (Compare-And-Swap)** for lock-free operations where possible
- Converts heavily contended bins to **red-black trees** (like HashMap)

```java
// Simplified ConcurrentHashMap.Node
static class Node<K, V> implements Map.Entry<K, V> {
    volatile V value;  // volatile for visibility
    volatile Node<K, V> next;  // volatile for visibility
    // ...
}
```

## Key Differences Summary

| Feature | HashMap | ConcurrentHashMap |
|---------|---------|-------------------|
| **Thread Safety** | No | Yes |
| **Null Keys/Values** | Allowed (one null key) | Not allowed |
| **Locking** | None | Fine-grained (per-bin) |
| **Read Performance** | Fast (no locks) | Fast (lock-free reads) |
| **Write Performance** | Fast (no locks) | Slightly slower (locking overhead) |
| **Concurrency** | None | High (multiple threads) |
| **Iterator** | Fail-fast | Weakly consistent |
| **Null handling** | `get(null)` works | `get(null)` throws NPE |
| **Size/isEmpty** | Not reliable in concurrent context | Reliable (snapshot-based) |

## ConcurrentHashMap Iterator: Weakly Consistent

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1);
map.put("b", 2);

// Iterator reflects the state of the map at some point during iteration
// It won't throw ConcurrentModificationException even if the map is modified
Iterator<Map.Entry<String, Integer>> it = map.entrySet().iterator();
while (it.hasNext()) {
    Map.Entry<String, Integer> entry = it.next();
    map.put("c", 3);  // Safe to modify during iteration
}
```

## Performance Comparison

### Read-Heavy Workload
```java
// ConcurrentHashMap is nearly as fast as HashMap for reads
// because reads are lock-free
```

### Write-Heavy Workload
```java
// ConcurrentHashMap has some overhead due to locking
// But still much better than Collections.synchronizedMap()
// because only the affected bin is locked
```

## When to Use Each

### Use `HashMap` when:
- Single-threaded environment
- Or you can guarantee external synchronization
- Maximum performance is needed (no locking overhead)

### Use `ConcurrentHashMap` when:
- Multiple threads access the map concurrently
- You need thread-safe operations without external synchronization
- Read-heavy workload (reads are lock-free)
- You need weakly consistent iterators

### Use `Collections.synchronizedMap()` when:
- You need a quick fix for thread safety
- Low concurrency (few threads)
- You can manually synchronize iteration

## Code Example: Producer-Consumer with ConcurrentHashMap

```java
public class ConcurrentCache {
    private final ConcurrentHashMap<String, String> cache = new ConcurrentHashMap<>();

    public String get(String key) {
        return cache.get(key);  // Lock-free read
    }

    public void put(String key, String value) {
        cache.put(key, value);  // Fine-grained write lock
    }

    // Atomic operation — no race condition
    public String computeIfAbsent(String key, Function<String, String> loader) {
        return cache.computeIfAbsent(key, loader);
    }
}
```

## Common Interview Questions

### Q: Why doesn't ConcurrentHashMap allow null keys/values?
A: Because `null` is ambiguous in a concurrent context — you can't distinguish between "key not found" and "key maps to null value" without additional synchronization.

### Q: What is a "weakly consistent" iterator?
A: An iterator that reflects the state of the map at some point during iteration, but doesn't throw `ConcurrentModificationException` if the map is modified during iteration.

### Q: How does ConcurrentHashMap achieve high concurrency?
A: By using fine-grained locking (per-bin in Java 8+) and lock-free reads (using volatile variables and CAS operations).

## Key Takeaway

> `ConcurrentHashMap` is the go-to choice for thread-safe maps in Java. It provides high concurrency with minimal performance overhead compared to `HashMap`, and its weakly consistent iterators make it safe to use in concurrent environments without external synchronization.
