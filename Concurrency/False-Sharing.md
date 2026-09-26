# False Sharing in Multithreading

## What is False Sharing?

False sharing occurs when multiple threads modify different variables that share the same cache line, causing unnecessary cache coherence traffic and performance degradation.

## How It Works

### Cache Line Structure
- CPU caches data in cache lines (typically 64 bytes)
- Each cache line can hold multiple variables
- When one variable in a cache line is modified, the entire line is invalidated in other caches

### Example
```java
// BAD - variables share cache line
class BadCounter {
    public long counter1;
    public long counter2;
}

// Thread 1 increments counter1
// Thread 2 increments counter2
// Both variables in same cache line
// Each write invalidates the other's cache line
```

## How to Avoid False Sharing

### 1. **Padding**
```java
// GOOD - separate cache lines
class PaddedCounter {
    public long counter1;
    public long padding1, padding2, padding3, padding4; // 64 bytes padding
    
    public long counter2;
    public long padding5, padding6, padding7, padding8;
}
```

### 2. **@Contended Annotation (Java 8+)**
```java
@jdk.internal.vm.annotation.Contended
public long counter1;

@Contended
public long counter2;
```

### 3. **Use Atomic Classes**
```java
// Atomic classes handle padding internally
AtomicLong counter1 = new AtomicLong();
AtomicLong counter2 = new AtomicLong();
```

## When It Matters
- **High-contention scenarios** - many threads updating counters
- **Low-latency systems** - microseconds matter
- **Performance-critical applications** - trading systems, messaging

## Interview Tip
Explain that false sharing is a hardware-level issue. Mention that it's especially problematic in Java due to object layout and memory alignment.