# HashMap vs ConcurrentHashMap

## Core Differences

| Feature | HashMap | ConcurrentHashMap |
|---------|---------|------------------|
| **Thread Safety** | Not thread-safe | Thread-safe |
| **Synchronization** | None | Segment-level or CAS operations |
| **Null Keys** | Allowed (1 null key) | Not allowed |
| **Null Values** | Allowed | Not allowed |
| **Performance** | Better single-threaded | Slightly slower due to synchronization |
| **Iteration** | Fail-fast (ConcurrentModificationException) | Weakly consistent (no exception) |

## ConcurrentHashMap Evolution

### Java 7: Segment-based
- Array of segments (default 16)
- Each segment is a separate HashMap with its own lock
- Lock striping for better concurrency
- Multiple threads can operate on different segments simultaneously

### Java 8+: CAS + Synchronized
- No segments - uses synchronized blocks on individual nodes
- Uses CAS (Compare-And-Swap) operations for updates
- Better scalability and performance
- Treeified buckets for collision handling

## When to Use Each

### Use HashMap When:
- **Single-threaded access** - no concurrent modifications
- **Better performance needed** - no synchronization overhead
- **Null keys/values allowed** - need to store nulls
- **Local variables** - method-scoped maps

```java
// Single-threaded context
private Map<String, User> userCache = new HashMap<>();
```

### Use ConcurrentHashMap When:
- **Multiple threads accessing** - concurrent read/write
- **Shared data structures** - across threads
- **High concurrency** - need thread-safe operations
- **Null not allowed** - acceptable constraint

```java
// Shared across threads
private final Map<String, Object> sharedConfig = new ConcurrentHashMap<>();
```

## Important Operations

### Atomic Operations in ConcurrentHashMap
```java
// Atomic put-if-absent
sharedConfig.putIfAbsent(key, value);

// Atomic compute
sharedConfig.computeIfAbsent(key, k -> new Value());

// Atomic merge
sharedConfig.merge(key, value, (old, newVal) -> old + newVal);
```

## Interview Tip
Explain that ConcurrentHashMap provides better concurrency than Hashtable (which locks the entire map). Mention that iteration is weakly consistent - may or may not reflect concurrent modifications.