# Synchronized vs Volatile

## Core Differences

| Feature | Synchronized | Volatile |
|---------|-------------|----------|
| **Atomicity** | Yes (atomic block) | No (only visibility) |
| **Visibility** | Yes | Yes |
| **Ordering** | Yes | Yes |
| **Performance** | Slower (lock overhead) | Faster (no lock) |
| **Scope** | Methods/blocks | Variables only |
| **Mutual Exclusion** | Yes | No |

## When to Use Each

### Use Volatile When:
- **Single variable operations** - flag, counter
- **Visibility only needed** - no atomicity required
- **Performance critical** - avoid lock overhead
- **Read-heavy workloads** - frequent reads, occasional writes

```java
// Good for flags
private volatile boolean running = true;

while (running) {
    // Do work
}
```

### Use Synchronized When:
- **Multiple variables** - need atomicity across them
- **Complex operations** - compound actions
- **Mutual exclusion needed** - only one thread at a time
- **Read-modify-write cycles** - need atomicity

```java
// Need atomicity
private int counter = 0;

public synchronized void increment() {
    counter++; // Atomic with synchronized
}
```

## Volatile Limitations
```java
private volatile int count = 0;

// NOT thread-safe!
count++; // Read-modify-write, not atomic
```

## Synchronized Benefits
- Atomic operations
- Memory visibility
- Mutual exclusion
- Prevents thread interference

## Interview Tip
Explain that volatile is not a replacement for synchronization - it only provides visibility. For atomic operations, you need synchronization or atomic classes.