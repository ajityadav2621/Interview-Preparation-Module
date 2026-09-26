# False Sharing in Multithreading

## What Is False Sharing?

**False sharing** is a performance problem in multi-threaded applications where threads on **different cores** write to **different variables** that happen to live on the **same CPU cache line**.

Even though the threads are logically independent, the CPU's cache coherency protocol (MESI) treats them as "sharing" — causing frequent cache line invalidations and performance degradation.

### Cache Line Basics
- CPUs don't read individual bytes — they read **cache lines** (typically 64 bytes)
- When one core writes to a cache line, all other cores with that line must **invalidate** their copy
- This triggers expensive cache misses on the next read

```
CPU Core 0          CPU Core 1
[L1 cache]          [L1 cache]
   |                    |
counterA=1    <--->   counterB=2   ← same cache line!
   |                    |
   v                    v
[RAM: 0x100: counterA | counterB | ...other 48 bytes...]
```

Both cores constantly "fight" over the cache line even though they're not actually sharing data.

---

## How to Detect False Sharing

### Symptoms
- Application uses multiple cores but doesn't scale linearly
- High CPU but low throughput
- `perf stat` shows high cache-miss rates
- Thread dumps show many threads "running" but doing little work

### Tools
```bash
# Linux perf — record cache misses
perf record -e cache-misses -e cache-references -g java MyApp
perf report

# Intel VTune — shows false sharing explicitly
vtune -collect memory-access MyApp

# Java Flight Recorder (JFR) — built-in
jcmd <pid> JFR.start duration=60s filename=rec.jfr
# Then analyze in JDK Mission Control
```

---

## How to Avoid False Sharing in Java

### Strategy 1: Padding (Manual)
```java
class Counter {
    // Pad before and after to push to separate cache line
    private long p1, p2, p3, p4, p5, p6, p7;   // 56 bytes
    private volatile long count = 0;
    private long p8, p9, p10, p11, p12, p13, p14; // 56 bytes
}
```

### Strategy 2: Contended Annotation (Java 9+, public API since JDK 11)
Use `@Contended` (originally JEP 142 in JDK 9; made available to non-JDK code in JDK 11):

```java
import jdk.internal.vm.annotation.Contended;

public class Counter {
    @Contended
    private volatile long count;
}
```
JVM automatically adds padding around the field. Requires `-XX:+UseContended` JVM flag.

> **Note:** In JDK 9–10, `@Contended` was in the internal `jdk.internal.vm.annotation` package and effectively JDK-only. Since **JDK 11**, it can be used in application code with appropriate `--add-opens`/`--add-exports` flags for `java.base/jdk.internal.vm.annotation`. In JDK 17+, the standard alternative is to use `jdk.internal.vm.annotation.Contended` only if your runtime module exports it; otherwise prefer manual padding or `LongAdder`.

### Strategy 3: Array Padding (Stripe Pattern)
```java
// Striped counters — each thread updates its own slot
public class StripedCounter {
    private static final int STRIPE_COUNT = Runtime.getRuntime().availableProcessors();
    
    // Padded array — each stripe in its own cache line
    private final long[] counts = new long[STRIPE_COUNT * 16];  // 16 longs per stripe (cache line ≈ 128 bytes)
    
    public void increment() {
        int stripe = ThreadLocalRandom.current().nextInt(STRIPE_COUNT);
        counts[stripe * 16]++;
    }
    
    public long sum() {
        long total = 0;
        for (int i = 0; i < STRIPE_COUNT; i++) {
            total += counts[i * 16];
        }
        return total;
    }
}
```

### Strategy 4: Reorganize Data Layout
Group fields by access pattern (hot/cold, read-mostly/write-mostly):

```java
// Bad: hot fields mixed with cold
class Order {
    long id;              // hot
    String notes;         // cold
    BigDecimal amount;    // hot
    String auditLog;      // cold
}

// Better: separate hot and cold
class Order {
    // Hot fields together
    long id;
    BigDecimal amount;
    long lastModified;
    
    // Padding if needed
    
    // Cold fields
    String notes;
    String auditLog;
}
```

### Strategy 5: Use ThreadLocal Where Possible
```java
private final ThreadLocal<Counter> threadLocalCounter = 
    ThreadLocal.withInitial(Counter::new);

// No contention at all — each thread has its own copy
```

---

## Real-World Impact

### AtomicLong vs LongAdder (Java 8+)

```java
// Single contended AtomicLong — high cache line bouncing
AtomicLong counter = new AtomicLong();
counter.incrementAndGet();

// LongAdder — striped internally, much better under contention
LongAdder adder = new LongAdder();
adder.increment();
long total = adder.sum();  // sum across stripes
```

`LongAdder` is essentially an array of `Cell` objects, each in its own cache line. Under high contention, it outperforms `AtomicLong` by **5-10x** or more.

### Disruptor's Sequence Pattern
The LMAX Disruptor library uses padded `Sequence` objects to avoid false sharing in its ring buffer — one of the reasons it can achieve 100M+ ops/sec.

```java
// Simplified Disruptor pattern
class PaddedSequence extends Sequence {
    public volatile long p0, p1, p2, p3, p4, p5, p6;  // pad before
    private volatile long value;
    public volatile long q0, q1, q2, q3, q4, q5, q6;  // pad after
}
```

---

## When to Worry About False Sharing

✅ **Worth optimizing:**
- High-throughput counters (metrics, sequence numbers)
- Lock-free data structures (queues, ring buffers)
- Hot fields in heavily multi-threaded objects
- High-performance trading / gaming / HPC

❌ **Don't bother:**
- Most business applications
- Code that doesn't show in profiling
- Premature optimization without measurement
- Fields updated by only one thread

---

## Quick Checklist

1. **Measure first** — profile to confirm false sharing
2. **Try `LongAdder`** before manual padding
3. **Add `@Contended`** if on JDK 11+
4. **Pad manually** for fine control
5. **Re-test** to confirm improvement

---

## Key Takeaway

> False sharing is a **silent performance killer** in low-latency, multi-threaded systems. Two threads writing to "different" variables can still cause 10x slowdown if those variables share a cache line. Profile first, then pad wisely using `@Contended`, stripe patterns, or careful data layout. Don't optimize without evidence.
