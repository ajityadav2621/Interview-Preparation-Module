# Memory Analysis & Production Troubleshooting Interview Questions

> **Note**: Some foundational topics (memory leak basics, common causes, ThreadLocal leaks, static collections, unbounded caches, listeners/callbacks) are covered in detail in [`memory-leak-gc.md`](memory-leak-gc.md). This file focuses on advanced topics: GC eligibility, heap dumps, dominator trees, JFR, production troubleshooting, and real-world scenarios.

---

## 1. What is a memory leak in Java if Garbage Collection is automatic?

A memory leak in Java occurs when objects are **still reachable** (referenced) but **no longer needed** by the application. The GC cannot reclaim them because they're still "alive" from the GC's perspective, even though the application will never use them again.

**Key insight**: Unlike C/C++ where memory leaks happen due to lost pointers, Java memory leaks happen due to **unintentional strong references** that keep objects alive indefinitely.

**Example**:
```java
public class LeakExample {
    private static final Map<String, byte[]> cache = new HashMap<>();
    
    public void process(String key) {
        cache.put(key, new byte[1024 * 1024]); // 1MB per entry
        // Entries are never removed — cache grows forever
    }
}
```

---

## 2. How does the JVM determine whether an object is eligible for Garbage Collection?

The JVM uses **reachability analysis** to determine if an object is eligible for GC:

1. **Start from GC Roots** (see next question)
2. **Trace references** through the object graph
3. If an object is **reachable** from any GC Root → it's **alive**
4. If an object is **not reachable** from any GC Root → it's **eligible** for GC

**Important**: An object becomes eligible only when:
- It is unreachable from GC Roots
- Its `finalize()` method (if any) has already been called (in older JVMs)

**Note**: The JVM does NOT guarantee when GC will run — only that it *will* run when memory is low.

---

## 3. What are GC Roots?

**GC Roots** are the starting points for reachability analysis. Objects reachable from GC Roots are considered alive. Common GC Roots include:

| GC Root Type | Examples |
|--------------|----------|
| **Local variables** | Method parameters, local variables in stack frames |
| **Active threads** | Live threads (including daemon threads) |
| **Static variables** | Classes loaded by system class loader |
| **JNI references** | Native code references |
| **System classes** | `java.lang.System`, `java.lang.Class` |
| **Thread stacks** | Objects referenced from thread stack frames |

**Visualization**:
```
GC Roots (Thread stacks, statics, etc.)
    │
    ├──► Object A ──► Object B ──► Object C  (all alive)
    │
    └──► Object D ──► Object E               (all alive)
    
    Object F ──► Object G                     (NOT reachable → eligible for GC)
```

---

## 4. What are the most common causes of memory leaks in Java?

1. **Static collections** holding references indefinitely
2. **ThreadLocal** variables not cleaned up (especially in app servers)
3. **Unclosed resources** (file handles, DB connections, streams)
4. **Listeners/callbacks** registered but never unregistered
5. **Inner classes** holding implicit references to outer classes
6. **Caches without eviction** policies
7. **String.intern()** in loops (older JVMs — PermGen/Metaspace leaks)
8. **Session objects** in web apps not invalidated

---

## 5. How can static collections cause memory leaks?

Static collections live as long as the JVM. If they accumulate objects without bounds:

```java
public class StaticLeak {
    // Lives for the entire JVM lifetime
    private static final List<Object> cache = new ArrayList<>();
    
    public static void add(Object obj) {
        cache.add(obj); // Never removed!
    }
}
```

**Impact**: Every object added stays in memory forever, even if no longer needed.

**Fix**: Use bounded collections with eviction:
```java
private static final Map<String, Object> cache = 
    new LinkedHashMap<String, Object>(100, 0.75f, true) {
        protected boolean removeEldestEntry(Map.Entry<String, Object> eldest) {
            return size() > 1000;
        }
    };
```

---

## 6. How can ThreadLocal cause a memory leak?

In application servers (Tomcat, Jetty), threads are pooled and reused across requests. `ThreadLocal` values are stored in the thread's `ThreadLocalMap`. When an app is undeployed:

- The thread pool persists
- `ThreadLocal` values are not automatically cleaned up
- The entire web application class loader (and all its classes) is leaked

**Fix**: Always clean up in `finally`:
```java
try {
    formatter.get().format(date);
} finally {
    formatter.remove(); // Critical!
}
```

---

## 7. How can unbounded caches cause memory problems?

Caches without size limits or eviction policies grow indefinitely:

```java
public class CacheLeak {
    private Map<String, Object> cache = new HashMap<>();
    
    public Object get(String key) {
        Object value = cache.get(key);
        if (value == null) {
            value = loadFromDatabase(key);
            cache.put(key, value); // Never evicted!
        }
        return value;
    }
}
```

**Impact**: Eventually triggers `OutOfMemoryError: Java heap space`.

**Fix**: Use proper caching libraries:
```java
private Cache<String, Object> cache = Caffeine.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build();
```

---

## 8. How can listeners and callbacks cause memory leaks?

Event sources that accumulate listeners without removal:

```java
public class EventSource {
    private List<EventListener> listeners = new ArrayList<>();
    
    public void addListener(EventListener listener) {
        listeners.add(listener);
        // Missing: removeListener()!
    }
}
```

**Impact**: Listeners (and their referenced objects) are never GC'd.

**Fix**: Provide `removeListener()` and ensure callers use it, or use `WeakReference` for listeners.

---

## 9. What is the difference between a memory leak and high memory usage?

| Aspect | Memory Leak | High Memory Usage |
|--------|-------------|-------------------|
| **Definition** | Objects unreachable by app but still referenced | Objects actively used by app |
| **GC behavior** | GC cannot reclaim | GC can reclaim if objects become unreachable |
| **Pattern** | Memory grows continuously, never decreases after GC | Memory fluctuates with load, stabilizes or decreases after GC |
| **Root cause** | Unintentional strong references | Legitimate high load or undersized heap |
| **Fix** | Remove unintended references | Increase heap, optimize usage, or scale |

**Key test**: If memory doesn't drop after a Full GC, you likely have a leak.

---

## 10. OutOfMemoryError vs StackOverflowError?

| Error | Cause | Where | Recovery |
|-------|-------|-------|----------|
| **`OutOfMemoryError`** | Heap or Metaspace exhaustion | Heap/Metaspace | Difficult — often requires heap dump analysis |
| **`StackOverflowError`** | Stack exhaustion (deep recursion) | Stack | Possible — fix recursion or increase stack size (`-Xss`) |

**StackOverflowError example**:
```java
public class StackOverflow {
    public static void recurse() {
        recurse(); // Infinite recursion
    }
}
```

**OutOfMemoryError example**:
```java
public class OOM {
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        while (true) {
            list.add(new byte[1024 * 1024]); // Allocate 1MB repeatedly
        }
    }
}
```

---

## 11. What is a Heap Dump?

A **heap dump** is a snapshot of all objects in the JVM heap at a specific point in time. It includes:

- All objects and their class types
- Object sizes and memory addresses
- Reference chains (which objects reference which)
- Primitive field values

**File format**: Binary `.hprof` file (JVM TI format)

**Use cases**:
- Diagnosing `OutOfMemoryError`
- Finding memory leaks
- Analyzing object retention
- Understanding memory distribution

---

## 12. How do you capture a Heap Dump from a production JVM?

### Automatic (on OOM):
```bash
# JVM flag — auto-dump on OutOfMemoryError
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps/
```

### Manual (live JVM):
```bash
# Using jmap (may pause the JVM briefly)
jmap -dump:format=b,file=heap.hprof <pid>

# Using jcmd (preferred, less intrusive)
jcmd <pid> GC.heap_dump /path/to/heap.hprof

# Using JMX programmatically
HotSpotDiagnosticMXBean mxBean = ManagementFactory.newPlatformMXBeanProxy(
    ManagementFactory.getPlatformMBeanServer(),
    "com.sun.management:type=HotSpotDiagnostic",
    HotSpotDiagnosticMXBean.class
);
mxBean.dumpHeap("/path/to/heap.hprof", true);
```

### Kubernetes considerations:
```bash
# Exec into the pod
kubectl exec -it <pod-name> -- jcmd 1 GC.heap_dump /tmp/heap.hprof

# Copy the dump to local machine
kubectl cp <pod-name>:/tmp/heap.hprof ./heap.hprof
```

---

## 13. How do you analyze a Heap Dump?

### Tools:
1. **Eclipse MAT** (Memory Analyzer Tool) — most popular, free
2. **VisualVM** — lightweight, bundled with JDK
3. **JProfiler** — commercial, powerful
4. **YourKit** — commercial, excellent UI
5. **jhat** — basic, bundled with JDK (deprecated)

### Analysis steps in Eclipse MAT:
1. **Open** the `.hprof` file
2. **Run** automatic leak analysis (Leak Suspects Report)
3. **Check** the Dominator Tree for largest retained heaps
4. **Inspect** the Histogram for object counts
5. **Trace** reference chains to find why objects are retained

### Key views:
- **Histogram**: Class name vs. instance count/size
- **Dominator Tree**: Objects retaining the most memory
- **Path to GC Roots**: Why a specific object is not collectible
- **Leak Suspects**: Automated detection of common leak patterns

---

## 14. What is a Dominator Tree?

A **Dominator Tree** shows which objects dominate (retain) other objects in memory.

**Definition**: Object A **dominates** Object B if every path from a GC Root to B must go through A.

**Example**:
```
GC Root
  └──► Application (dominates)
        ├──► Session (dominates)
        │     └──► User (dominates)
        │           └──► Order (dominates)
        │                 └──► OrderItem
        └──► Cache (dominates)
              └──► CachedData
```

**Why it matters**: The Dominator Tree helps identify the **biggest memory consumers**. If `Application` is the top dominator, fixing leaks in `Session` or `Cache` will free the most memory.

---

## 15. How do you identify objects consuming most of the heap?

### Step 1: Check the Histogram
```
Class Name                     # Instances    Total Size
------------------------------------------------------
byte[]                         15,234        48,540 KB
java.lang.String               125,432       12,543 KB
com.app.Order                  8,921         8,921 KB
...
```

### Step 2: Drill into suspicious classes
- Right-click → **List Objects** → **With Incoming References**
- Check **Retained Heap** (not just Shallow Heap)

### Step 3: Trace to GC Roots
- Right-click → **Path to GC Roots** → **Exclude Weak/Soft References**
- This shows why the object is retained

### Step 4: Analyze the Dominator Tree
- Sort by **Retained Heap** (descending)
- The top entries are your biggest memory consumers

---

## 16. What is a Retained Heap?

**Retained Heap** = The amount of memory that would be freed if this object (and everything it dominates) were garbage collected.

| Metric | Definition |
|--------|------------|
| **Shallow Heap** | Memory occupied by the object itself |
| **Retained Heap** | Shallow Heap + all objects reachable only through this object |

**Example**:
```java
class Order {
    List<OrderItem> items;  // Retained heap includes all OrderItems
    Customer customer;      // Retained heap includes Customer
}
```

If `Order` is the only reference to its `items` and `customer`, then removing `Order` frees all of them.

**Why it matters**: Shallow heap can be misleading. A `byte[]` might have small shallow heap but huge retained heap if it dominates many objects.

---

## 17. Heap Dump vs Thread Dump?

| Aspect | Heap Dump | Thread Dump |
|--------|-----------|-------------|
| **Content** | All objects in heap | All thread states and stack traces |
| **File size** | Large (hundreds of MB to GB) | Small (text, ~100KB) |
| **Use case** | Memory leaks, OOM | Deadlocks, CPU spikes, hangs |
| **Format** | Binary `.hprof` | Text (platform-specific) |
| **Impact** | Brief pause during capture | Brief pause during capture |

**When to use which**:
- **Heap Dump**: `OutOfMemoryError`, suspected memory leak, high memory usage
- **Thread Dump**: Application hang, deadlock, high CPU, thread contention

**Capture thread dump**:
```bash
jstack <pid> > thread_dump.txt
jcmd <pid> Thread.print
kill -3 <pid>  # Linux — prints to stdout
```

---

## 18. What tools do you use to investigate Java memory issues?

### Built-in JDK Tools:
| Tool | Purpose |
|------|---------|
| `jstat` | Monitor GC activity in real-time |
| `jmap` | Capture heap dumps, view heap summary |
| `jcmd` | Modern replacement for jmap/jstack |
| `jhat` | Basic heap dump analysis (deprecated) |
| `jstack` | Capture thread dumps |
| `jinfo` | View/modify JVM flags |
| `jconsole` | GUI for JMX monitoring |
| `visualvm` | GUI profiling and monitoring |

### Third-party Tools:
| Tool | Purpose |
|------|---------|
| **Eclipse MAT** | Deep heap dump analysis |
| **JProfiler** | Commercial profiler |
| **YourKit** | Commercial profiler |
| **GCViewer** | GC log analysis |
| **GCEasy** | Online GC log analysis |
| **HeapHero** | Online heap dump analysis |

### APM Tools:
- **New Relic**, **Dynatrace**, **AppDynamics** — production monitoring with memory profiling

---

## 19. How does Java Flight Recorder help with memory analysis?

**Java Flight Recorder (JFR)** is a low-overhead profiling tool built into the JVM (OpenJDK 11+).

### Memory-related events JFR captures:
- **Object allocation** in new TLAB (Thread-Local Allocation Buffer)
- **Object allocation outside TLAB**
- **GC events** (all phases)
- **Metaspace allocation**
- **Class loading/unloading**
- **Thread start/stop**

### How to use JFR for memory analysis:

```bash
# Start recording (JDK 11+)
jcmd <pid> JFR.start name=recording settings=profile duration=60s filename=recording.jfr

# Or start with JVM flags
-XX:StartFlightRecording=duration=60s,filename=recording.jfr
```

### Analysis in Java Mission Control (JMC):
1. **Memory** page → view allocation stacks
2. **GC** page → view GC pauses and causes
3. **Allocations** tab → see which methods allocate the most
4. **Object Statistics** → see class-wise allocation counts

### Advantages over heap dumps:
- **Low overhead** (~1-2% CPU)
- **Time-series data** — see allocation patterns over time
- **No need to trigger OOM** — can profile production safely
- **Shows allocation stacks** — not just object counts

---

## 20. How can excessive object creation affect application performance?

### Problems:
1. **GC pressure**: More objects → more frequent GC → longer pauses
2. **Allocation overhead**: Even cheap allocations add up
3. **Cache pollution**: Young Gen fills faster → promotes objects prematurely
4. **TLAB exhaustion**: Threads compete for allocation space

### Example — String concatenation in loops:
```java
// BAD: Creates many intermediate String objects
String result = "";
for (String s : list) {
    result += s; // Each iteration creates a new String
}

// GOOD: Single StringBuilder
StringBuilder sb = new StringBuilder();
for (String s : list) {
    sb.append(s);
}
String result = sb.toString();
```

### Example — Autoboxing:
```java
// BAD: Creates 1M Integer objects
Long sum = 0L;
for (int i = 0; i < 1_000_000; i++) {
    sum += i; // Autoboxing creates Long objects
}

// GOOD: Use primitive long
long sum = 0L;
for (int i = 0; i < 1_000_000; i++) {
    sum += i;
}
```

### Mitigation:
- Use primitives instead of wrappers
- Pre-size collections
- Reuse objects (object pools for expensive objects)
- Use `StringBuilder` for string concatenation
- Avoid creating objects in hot loops

---

## 21. How can String objects contribute to memory pressure?

### String Pool (String Interning):
```java
String s1 = "hello";        // In pool
String s2 = new String("hello"); // NOT in pool (unless interned)
String s3 = s2.intern();    // Now in pool
```

### Memory pressure sources:
1. **Large strings**: Each `String` has `char[]` (2 bytes per char in Java 8+, 1-2 bytes in Java 9+)
2. **String.intern()**: In Java 7+, interned strings are in the heap (PermGen → Metaspace change)
3. **Substrings** (Java 7+): No longer share the parent's `char[]`, but still create new objects
4. **String concatenation**: Creates intermediate `StringBuilder` and `String` objects

### Example — Large string memory:
```java
// A 1MB string
String large = new String(new char[1024 * 1024]); // ~2MB in Java 8 (char[])

// In Java 9+, compact strings:
// - If all chars < 256, uses byte[] (1 byte per char)
// - Otherwise uses byte[] with UTF-16 encoding
```

### Best practices:
- Use `StringBuilder` for concatenation
- Be cautious with `String.intern()` in loops
- Consider `byte[]` for large binary data instead of `String`

---

## 22. How can Hibernate/JPA cause unexpected memory growth?

### Common causes:

1. **Session/EntityManager cache (1st-level cache)**:
   - All loaded entities stay in the persistence context
   - Long-running sessions accumulate entities

2. **N+1 query problem**:
   - Lazy loading triggers many small queries
   - Each query creates entity objects

3. **Batching without flushing**:
   - Large batch inserts accumulate in session
   - Memory grows until flush/clear

4. **Second-level cache misconfiguration**:
   - Unbounded cache stores all entities
   - No eviction policy

5. **Fetch type issues**:
   - `EAGER` fetching loads unnecessary associations
   - Creates large object graphs

### Example — Session not cleared:
```java
// BAD: Session grows indefinitely
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

for (int i = 0; i < 100000; i++) {
    session.save(entity); // All entities stay in session!
}
tx.commit();
session.close(); // Memory spike until close

// GOOD: Clear session periodically
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

for (int i = 0; i < 100000; i++) {
    session.save(entity);
    if (i % 1000 == 0) {
        session.flush();
        session.clear(); // Free memory
    }
}
tx.commit();
session.close();
```

### Fixes:
- Use `session.clear()` or `entityManager.clear()` in batch operations
- Use `ScrollableResults` for large data processing
- Configure 2nd-level cache with eviction (size/time-based)
- Use `JOIN FETCH` to avoid N+1
- Keep persistence contexts short-lived

---

## 23. How can an incorrectly configured cache cause OutOfMemoryError?

### Problem scenarios:

1. **Unbounded cache**:
   ```java
   // BAD: No size limit
   Cache<String, Object> cache = Caffeine.newBuilder().build();
   ```

2. **Wrong eviction policy**:
   ```java
   // BAD: Time-based only, but access pattern is bursty
   Cache<String, Object> cache = Caffeine.newBuilder()
       .expireAfterAccess(1, TimeUnit.HOURS)
       .build(); // No size limit!
   ```

3. **Cache storing large values**:
   ```java
   // BAD: Caching entire result sets
   cache.put("query:users", userRepository.findAll()); // Could be MBs
   ```

4. **Cache key leaks**:
   ```java
   // BAD: Keys are complex objects with many references
   cache.put(new ComplexKey(...), value);
   ```

### Example — OOM from cache:
```java
// Application runs for days, cache grows unbounded
// Eventually: java.lang.OutOfMemoryError: Java heap space
```

### Fixes:
```java
// GOOD: Bounded cache with both size and time eviction
Cache<String, Object> cache = Caffeine.newBuilder()
    .maximumSize(10_000)                    // Hard limit
    .expireAfterWrite(30, TimeUnit.MINUTES) // Time limit
    .recordStats()                          // Monitor hit rate
    .build();
```

---

## 24. How would you investigate continuously increasing Old Generation usage?

### Symptoms:
- Young GC is normal (frequent, fast)
- Old Gen usage grows steadily after each Full GC
- Eventually triggers Full GC or OOM

### Investigation steps:

1. **Check GC logs**:
   ```bash
   -Xlog:gc*:file=gc.log:time,level,tags
   ```
   Look for: Old Gen usage before/after Full GC

2. **Capture heap dump** during high Old Gen usage:
   ```bash
   jcmd <pid> GC.heap_dump /tmp/old_gen.hprof
   ```

3. **Analyze in Eclipse MAT**:
   - Open Dominator Tree
   - Sort by Retained Heap
   - Identify classes with high instance counts in Old Gen

4. **Common causes**:
   - **Premature promotion**: Young objects surviving too few GC cycles
   - **Large objects**: Directly allocated in Old Gen (TLAB overflow)
   - **Memory leak**: Objects unintentionally retained
   - **Insufficient Young Gen**: Objects promoted before natural death

5. **Check GC tuning**:
   ```bash
   # Increase Young Gen to allow more scavenging
   -Xmn512m  # 512MB Young Gen
   
   # Or use ratio
   -XX:NewRatio=3  # Young:Old = 1:3
   ```

---

## 25. Why can memory remain high even after Garbage Collection?

### Reasons:

1. **Live data is genuinely high**:
   - Application legitimately needs that much memory
   - Caches, sessions, loaded data are all "in use"

2. **Memory leak**:
   - Objects are reachable but not needed
   - GC cannot reclaim them

3. **GC didn't run**:
   - `System.gc()` is just a suggestion
   - GC runs based on allocation pressure, not on demand

4. **Wrong GC area inspected**:
   - Checking heap usage immediately after minor GC
   - Old Gen only shrinks after Full GC

5. **Metaspace/PermGen growth**:
   - Class metadata accumulates
   - Not affected by heap GC

### How to verify:
```bash
# Trigger Full GC and check
jcmd <pid> GC.run

# Then check memory
jcmd <pid> GC.heap_info
```

If memory drops after Full GC → live data is high (not a leak).
If memory doesn't drop → likely a memory leak.

---

## 26. How would you troubleshoot frequent Full GC?

### Symptoms:
- Application pauses every few seconds/minutes
- `Full GC` entries in GC logs
- High CPU usage during GC

### Investigation steps:

1. **Enable GC logging**:
   ```bash
   -Xlog:gc*:file=gc.log:time,level,tags
   ```

2. **Analyze GC logs**:
   ```bash
   # Look for patterns
   [Full GC (Ergonomics) ... 0.123s]
   ```
   - Frequency: How often?
   - Cause: Ergonomics? Allocation Failure? System.gc()?
   - Duration: How long does each Full GC take?

3. **Common causes**:
   - **Old Gen too small**: Objects promoted faster than collected
   - **Metaspace too small**: Class metadata pressure
   - **System.gc() calls**: Explicit GC requests
   - **Heap fragmentation**: Especially with CMS
   - **Promotion failure**: Young Gen can't fit in Old Gen

4. **Fixes**:
   ```bash
   # Increase heap
   -Xmx4g
   
   # Increase Old Gen
   -XX:MaxRAMPercentage=75.0
   
   # Disable explicit GC
   -XX:+DisableExplicitGC
   
   # Use G1GC (better for large heaps)
   -XX:+UseG1GC
   
   # Or ZGC/Shenandoah for very low pause times
   -XX:+UseZGC
   ```

5. **Monitor with jstat**:
   ```bash
   jstat -gc <pid> 1s
   ```
   Watch: F (Full GC count), FGCT (Full GC time)

---

## 27. How would you distinguish a memory leak from insufficient heap size?

### Test: Force Full GC and observe

```bash
# Trigger Full GC
jcmd <pid> GC.run

# Check memory before and after
jcmd <pid> GC.heap_info
```

### Decision matrix:

| Observation | Diagnosis | Action |
|-------------|-----------|--------|
| Memory drops significantly after Full GC | **Insufficient heap** | Increase `-Xmx` |
| Memory barely changes after Full GC | **Memory leak** | Find and fix leak |
| Memory grows linearly over time | **Memory leak** | Analyze heap dump |
| Memory fluctuates with load, stabilizes | **Normal** | May need tuning |
| Old Gen grows, Young Gen normal | **Leak or promotion issue** | Check object lifetimes |

### Additional checks:
1. **Heap histogram over time**:
   ```bash
   jmap -histo <pid> > histo1.txt
   # ... wait ...
   jmap -histo <pid> > histo2.txt
   # Compare — if same classes keep growing, it's a leak
   ```

2. **Monitor with JMX**:
   ```java
   MemoryUsage heap = memoryBean.getHeapMemoryUsage();
   long used = heap.getUsed();
   long max = heap.getMax();
   // Track over time
   ```

---

## 28. How would you troubleshoot OutOfMemoryError in a Spring Boot application?

### Step 1: Enable OOM dump
```yaml
# application.properties or JVM flags
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dumps/
-XX:+ExitOnOutOfMemoryError  # Optional: crash fast
```

### Step 2: Collect context
```bash
# GC logs
-Xlog:gc*:file=gc.log:time,level,tags

# Thread dump at OOM
jstack <pid> > thread_dump.txt

# System info
jinfo <pid> > jinfo.txt
jcmd <pid> VM.flags > flags.txt
```

### Step 3: Analyze heap dump
1. Open in Eclipse MAT
2. Run Leak Suspects Report
3. Check Dominator Tree
4. Identify top memory consumers

### Step 4: Common Spring Boot causes
- **Request-scoped beans** not released
- **Large request/response bodies** held in memory
- **Undelivered Kafka/RabbitMQ messages** accumulating
- **Scheduled tasks** creating objects without cleanup
- **@Cacheable** with unbounded cache
- **Hibernate Session** not cleared in batch operations

### Step 5: Fix and verify
```java
// Example fix — clear Hibernate session in batch
@Transactional
public void processLargeDataset(List<Data> items) {
    int batchSize = 1000;
    for (int i = 0; i < items.size(); i++) {
        entityManager.persist(items.get(i));
        if (i % batchSize == 0) {
            entityManager.flush();
            entityManager.clear();
        }
    }
}
```

---

## 29. How would you investigate memory issues in a Java application running inside Kubernetes?

### Step 1: Check container limits
```bash
kubectl describe pod <pod-name>
# Look for:
#   Limits:
#     memory:  2Gi
#   Requests:
#     memory:  1Gi
```

### Step 2: Check actual memory usage
```bash
kubectl top pod <pod-name>
# Shows current memory/CPU usage
```

### Step 3: Check for OOMKilled
```bash
kubectl get pod <pod-name>
# STATUS: OOMKilled (exit code 137)
```

### Step 4: Capture heap dump from K8s
```bash
# Exec into pod
kubectl exec -it <pod-name> -- /bin/bash

# Find Java process
ps aux | grep java

# Capture heap dump
jcmd <pid> GC.heap_dump /tmp/heap.hprof

# Copy to local machine
kubectl cp <pod-name>:/tmp/heap.hprof ./heap.hprof
```

### Step 5: Check JVM memory settings
```bash
# Inside pod
java -XX:+PrintFlagsFinal -version | grep -E 'MaxHeapSize|MaxRAM'
```

**Important**: In containers, JVM may not detect container memory limits correctly (pre-Java 10). Use:
```bash
-XX:+UseContainerSupport  # Java 10+
-XX:MaxRAMPercentage=75.0  # Use 75% of container memory
```

### Step 6: Common K8s-specific issues
- **Memory limits too low**: JVM heap + metaspace + native memory > container limit
- **No memory limits**: Pod consumes all node memory, gets OOMKilled by kubelet
- **Multiple JVMs in one pod**: Memory contention
- **Sidecar containers**: Competing for memory

---

## 30. A production application starts with 2 GB memory but gradually reaches 8 GB and crashes. How would you identify the root cause?

### Scenario analysis:
- **Start**: 2 GB (normal)
- **Growth**: Gradual increase to 8 GB
- **Crash**: `OutOfMemoryError` or OOMKilled

### Investigation plan:

#### Phase 1: Confirm it's a leak (not just high load)
```bash
# Monitor memory over time
kubectl top pod <pod> --containers=''

# Check if memory drops after restart
# If it grows again → likely a leak
```

#### Phase 2: Capture evidence
```bash
# 1. Enable heap dump on OOM (if not already)
-XX:+HeapDumpOnOutOfMemoryError

# 2. Capture heap dump before crash (if possible)
jcmd <pid> GC.heap_dump /tmp/leak.hprof

# 3. Capture GC logs
-Xlog:gc*:file=gc.log:time,level,tags

# 4. Capture thread dump
jstack <pid> > thread_dump.txt
```

#### Phase 3: Analyze heap dump
1. **Open in Eclipse MAT**
2. **Check Leak Suspects Report**
3. **Examine Dominator Tree** — what's retaining memory?
4. **Compare histograms** over time (if multiple dumps available)

#### Phase 4: Identify the leak source

**Common patterns for gradual growth**:

| Pattern | Likely Cause | Investigation |
|---------|--------------|---------------|
| `java.util.HashMap` growing | Static map without eviction | Check static collections |
| `byte[]` or `char[]` growing | Unbounded cache or buffer | Check cache implementations |
| `com.app.Session` growing | Session not invalidated | Check web session management |
| `java.lang.String` growing | String concatenation or interning | Check string handling |
| `com.app.Order` growing | Batch processing not clearing | Check batch jobs |

#### Phase 5: Trace reference chains
In Eclipse MAT:
1. Find the leaking class
2. Right-click → **Path to GC Roots**
3. Follow the chain back to the root
4. Identify the code path that creates the leak

#### Phase 6: Fix and validate
- Fix the identified leak
- Deploy to staging
- Run load test
- Monitor memory for 24-48 hours
- Verify memory stabilizes

### Example finding:
```
Dominator Tree:
1. com.app.Session @ 0x12345 — Retained: 3.2 GB
   └── com.app.User @ 0x12346 — Retained: 2.8 GB
       └── java.util.ArrayList @ 0x12347 — Retained: 2.5 GB
           └── com.app.Order @ 0x12348 — Retained: 2.0 GB
               └── com.app.OrderItem @ ... — Retained: 1.8 GB

Root cause: Sessions never expire, accumulating orders indefinitely.
Fix: Add session timeout and cleanup job.
```

---

## Quick Reference: Memory Investigation Checklist

```
□ Enable GC logging: -Xlog:gc*:file=gc.log
□ Enable heap dump on OOM: -XX:+HeapDumpOnOutOfMemoryError
□ Monitor with jstat: jstat -gc <pid> 1s
□ Check heap usage: jcmd <pid> GC.heap_info
□ Capture heap dump: jcmd <pid> GC.heap_dump heap.hprof
□ Analyze in Eclipse MAT: Dominator Tree, Leak Suspects
□ Check for: static collections, ThreadLocal, unclosed resources
□ Review: cache configurations, session management, batch processing
□ Compare: histograms over time to find growing object types
□ Trace: reference chains to find retention paths
```

---

## Related Files

- [`memory-leak-gc.md`](memory-leak-gc.md) — Memory leak fundamentals and common causes
- [`heap-vs-stack.md`](heap-vs-stack.md) — Heap vs Stack memory, OOM vs StackOverflow
- [`full-gc.md`](full-gc.md) — Full GC behavior and latency impact
- [`jvm-metrics.md`](../jvm-performance/jvm-metrics.md) — JVM monitoring metrics
- [`memory-coding-questions.md`](coding/memory-coding-questions.md) — Coding exercises
