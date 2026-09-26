# What Happens During a Full GC, and Why Can It Affect API Latency?

## What is a Full GC?

A **Full GC** (Garbage Collection) is a garbage collection event that **stops all application threads** and collects garbage from the **entire heap** — both the Young Generation and the Old Generation.

This is also called a **Stop-The-World (STW)** event because all application threads are paused until the GC completes.

## The GC Process

### Young Generation GC (Minor GC)

```
Before Minor GC:
┌─────────────────────────────────────────────┐
│  Young Gen:                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │  Eden    │ │ Survivor1│ │ Survivor2│    │
│  │  [A][B]  │ │  [C]     │ │  []      │    │
│  └──────────┘ └──────────┘ └──────────┘    │
│  Old Gen:                                   │
│  ┌──────────┐                               │
│  │  [D][E]  │                               │
│  └──────────┘                               │
└─────────────────────────────────────────────┘

After Minor GC (A, B collected, C moved to Survivor1):
┌─────────────────────────────────────────────┐
│  Young Gen:                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │  Eden    │ │ Survivor1│ │ Survivor2│    │
│  │  []      │ │  [C]     │ │  []      │    │
│  └──────────┘ └──────────┘ └──────────┘    │
│  Old Gen:                                   │
│  ┌──────────┐                               │
│  │  [D][E]  │                               │
│  └──────────┘                               │
└─────────────────────────────────────────────┘
```

- **Fast**: Only scans the Young Generation (typically 10-50ms)
- **Frequent**: Happens when Eden fills up
- **Low impact**: Only short-lived objects are collected

### Full GC (Major GC + Young GC)

```
Before Full GC:
┌─────────────────────────────────────────────┐
│  Young Gen:                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │  Eden    │ │ Survivor1│ │ Survivor2│    │
│  │  [A][B]  │ │  [C]     │ │  [D]     │    │
│  └──────────┘ └──────────┘ └──────────┘    │
│  Old Gen:                                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │  [E][F]  │ │  [G][H]  │ │  [I]     │    │
│  └──────────┘ └──────────┘ └──────────┘    │
└─────────────────────────────────────────────┘

After Full GC (E, F, G, H, I collected if unreachable):
┌─────────────────────────────────────────────┐
│  Young Gen:                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │  Eden    │ │ Survivor1│ │ Survivor2│    │
│  │  []      │ │  []      │ │  []      │    │
│  └──────────┘ └──────────┘ └──────────┘    │
│  Old Gen:                                   │
│  ┌──────────┐                               │
│  │  [E][G]  │  ← Only reachable objects    │
│  └──────────┘                               │
└─────────────────────────────────────────────┘
```

- **Slow**: Scans the entire heap (can take seconds)
- **Infrequent**: Happens when Old Generation fills up
- **High impact**: All application threads are paused

## Why Full GC Affects API Latency

### 1. Stop-The-World Pauses

During a Full GC, **all application threads are paused**. No requests can be processed, no responses can be sent.

```
Timeline:
Time: 0ms    100ms   200ms   300ms   400ms   500ms
      |       |       |       |       |       |
      Request1 arrives
              |       |       |       |       |
              Request2 arrives
                      |       |       |       |
                      Request3 arrives
                              |       |       |
                              Full GC starts (ALL THREADS PAUSED)
                              |       |       |
                              Full GC ends (2 seconds!)
                                      |       |
                                      Request1 responds (2s delay!)
                                      Request2 responds (2s delay!)
                                      Request3 responds (2s delay!)
```

### 2. Duration Scales with Heap Size

The larger the heap, the longer the Full GC pause:

| Heap Size | Typical Full GC Pause |
|-----------|----------------------|
| 1 GB | 100-200ms |
| 4 GB | 500-1000ms |
| 8 GB | 1-3 seconds |
| 16 GB | 3-10 seconds |
| 32 GB | 5-30 seconds |

### 3. Compacting Phase

After marking and sweeping, the GC must **compact** the heap — moving all live objects to one end to eliminate fragmentation. This is the most time-consuming part.

## GC Algorithms and Full GC Behavior

### Serial GC (Default for client applications)

```bash
-XX:+UseSerialGC
```

- Single-threaded GC
- Simplest algorithm
- Longest pause times
- Best for small heaps (< 100MB)

### Parallel GC (Default for server applications)

```bash
-XX:+UseParallelGC
```

- Multi-threaded GC (uses multiple CPU cores)
- Still has STW pauses for Full GC
- Good throughput, moderate pause times

### CMS (Concurrent Mark Sweep) — Deprecated in Java 14

```bash
-XX:+UseConcMarkSweepGC
```

- Minimizes STW pauses
- Most work done concurrently with application threads
- **Still has Full GC with STW pauses** when concurrent mode fails

### G1 (Garbage First) — Default since Java 9

```bash
-XX:+UseG1GC
```

- Designed for large heaps
- Divides heap into regions
- Can do **concurrent compaction**
- Still has STW pauses, but they're shorter and more predictable

### ZGC / Shenandoah — Low-Latency GC

```bash
-XX:+UseZGC           # Java 11+
-XX:+UseShenandoahGC  # Java 12+ (not in OpenJDK)
```

- **Sub-millisecond pause times**
- All work done concurrently
- Very low latency, but higher CPU overhead

## How to Diagnose Full GC Issues

### 1. Enable GC Logging

```bash
# Java 9+
-Xlog:gc*:gc.log:time

# Java 8
-XX:+PrintGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-Xloggc:gc.log
```

### 2. Sample GC Log Output

```
[GC (Allocation Failure) [PSYoungGen: 65536K->10752K(76288K)] 65536K->10760K(251904K), 0.005s]
[Full GC (Ergonomics) [PSYoungGen: 10752K->0K(76288K)] [OldGen: 181043K->10752K(175616K)] 191795K->10752K(251904K), [Metaspace: 2560K->2560K(1056768K)], 0.123s]
```

### 3. Monitor with jstat

```bash
jstat -gc <pid> 1s
# Shows GC frequency and pause times
```

### 4. Monitor with JMX

```java
List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
for (GarbageCollectorMXBean gcBean : gcBeans) {
    System.out.println(gcBean.getName() + ": " + 
        gcBean.getCollectionCount() + " collections, " +
        gcBean.getCollectionTime() + "ms total time");
}
```

## How to Reduce Full GC Impact

### 1. Choose the Right GC Algorithm

```bash
# For low latency:
-XX:+UseZGC -Xmx8g

# For throughput:
-XX:+UseParallelGC -Xmx8g

# For balanced:
-XX:+UseG1GC -Xmx8g
```

### 2. Tune Heap Sizes

```bash
# Set initial and max heap size
-Xms4g -Xmx4g

# Set young generation size
-Xmn1g

# Set G1 region size (for G1GC)
-XX:G1HeapRegionSize=16m
```

### 3. Reduce Object Allocation Rate

```java
// BAD: Creates many temporary objects
String result = "";
for (int i = 0; i < 1000; i++) {
    result += "item" + i;  // Creates new String each iteration
}

// GOOD: Use StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append("item").append(i);
}
String result = sb.toString();
```

### 4. Use Object Pooling

```java
// Reuse objects instead of creating new ones
public class ObjectPool<T> {
    private final Queue<T> pool = new ConcurrentLinkedQueue<>();
    private final Supplier<T> factory;

    public T borrow() {
        T obj = pool.poll();
        return obj != null ? obj : factory.get();
    }

    public void release(T obj) {
        pool.offer(obj);
    }
}
```

### 5. Tune GC Parameters

```bash
# Set target pause time (for G1GC)
-XX:MaxGCPauseMillis=200

# Set GC time ratio (for ParallelGC)
-XX:GCTimeRatio=19  # 5% of time spent in GC

# Enable concurrent GC threads
-XX:ConcGCThreads=4
```

## Key Takeaway

> Full GC pauses all application threads to collect garbage from the entire heap. The pause duration scales with heap size and can range from milliseconds to seconds. For latency-sensitive applications, choose low-pause GC algorithms (ZGC, Shenandoah), tune heap sizes, reduce object allocation rates, and monitor GC logs to identify and address Full GC issues.
