# JVM Performance Coding Questions

## 1. Monitor GC Programmatically

```java
import java.lang.management.*;
import java.util.List;

public class GCMonitor {
    private final List<GarbageCollectorMXBean> gcBeans;
    private long lastGcCount = 0;
    private long lastGcTime = 0;

    public GCMonitor() {
        this.gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
    }

    public void checkGcPressure() {
        long totalGcCount = 0;
        long totalGcTime = 0;

        for (GarbageCollectorMXBean gcBean : gcBeans) {
            totalGcCount += gcBean.getCollectionCount();
            totalGcTime += gcBean.getCollectionTime();
        }

        long gcCountDelta = totalGcCount - lastGcCount;
        long gcTimeDelta = totalGcTime - lastGcTime;

        if (gcCountDelta > 0) {
            System.out.printf("GC: %d collections, %dms in last interval%n",
                gcCountDelta, gcTimeDelta);
            
            if (gcTimeDelta > 1000) {
                System.err.println("WARNING: Long GC pause detected!");
            }
        }

        lastGcCount = totalGcCount;
        lastGcTime = totalGcTime;
    }

    public static void main(String[] args) {
        GCMonitor monitor = new GCMonitor();
        while (true) {
            monitor.checkGcPressure();
            try { Thread.sleep(5000); } catch (InterruptedException e) { break; }
        }
    }
}
```

---

## 2. Memory Usage Monitor

```java
import java.lang.management.*;

public class MemoryMonitor {
    private final MemoryMXBean memoryBean;
    private final List<GarbageCollectorMXBean> gcBeans;

    public MemoryMonitor() {
        this.memoryBean = ManagementFactory.getMemoryMXBean();
        this.gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
    }

    public void printMemoryStats() {
        MemoryUsage heap = memoryBean.getHeapMemoryUsage();
        MemoryUsage nonHeap = memoryBean.getNonHeapMemoryUsage();

        System.out.println("=== Memory Stats ===");
        System.out.printf("Heap: %d MB used / %d MB max (%.1f%%)%n",
            heap.getUsed() / 1024 / 1024,
            heap.getMax() / 1024 / 1024,
            (double) heap.getUsed() / heap.getMax() * 100);

        System.out.printf("Non-Heap: %d MB used / %d MB max%n",
            nonHeap.getUsed() / 1024 / 1024,
            nonHeap.getMax() / 1024 / 1024);

        // Check for memory pressure
        double heapUsage = (double) heap.getUsed() / heap.getMax();
        if (heapUsage > 0.8) {
            System.err.println("WARNING: Heap usage > 80%!");
        }
    }

    public void printGcStats() {
        System.out.println("=== GC Stats ===");
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            System.out.printf("%s: %d collections, %dms%n",
                gcBean.getName(),
                gcBean.getCollectionCount(),
                gcBean.getCollectionTime());
        }
    }
}
```

---

## 3. Thread Dump Analyzer

```java
import java.lang.management.*;
import java.util.*;

public class ThreadDumpAnalyzer {
    private final ThreadMXBean threadBean;

    public ThreadDumpAnalyzer() {
        this.threadBean = ManagementFactory.getThreadMXBean();
    }

    public void analyzeThreads() {
        long[] threadIds = threadBean.getAllThreadIds();
        ThreadInfo[] threadInfos = threadBean.getThreadInfo(threadIds, true, true);

        Map<Thread.State, Integer> stateCounts = new HashMap<>();
        int blockedCount = 0;
        int waitingCount = 0;

        for (ThreadInfo threadInfo : threadInfos) {
            if (threadInfo == null) continue;

            Thread.State state = threadInfo.getThreadState();
            stateCounts.merge(state, 1, Integer::sum);

            if (state == Thread.State.BLOCKED) {
                blockedCount++;
                System.out.printf("BLOCKED: %s (waiting for %s)%n",
                    threadInfo.getThreadName(),
                    threadInfo.getLockName());
            }

            if (state == Thread.State.WAITING || 
                state == Thread.State.TIMED_WAITING) {
                waitingCount++;
            }
        }

        System.out.println("=== Thread Analysis ===");
        System.out.println("Total threads: " + threadIds.length);
        System.out.println("Blocked threads: " + blockedCount);
        System.out.println("Waiting threads: " + waitingCount);
        System.out.println("State distribution: " + stateCounts);

        // Check for deadlocks
        long[] deadlocked = threadBean.findDeadlockedThreads();
        if (deadlocked != null) {
            System.err.println("DEADLOCK DETECTED! " + deadlocked.length + " threads deadlocked");
        }
    }
}
```

---

## 4. Detect Memory Leak Pattern

```java
import java.util.*;

public class MemoryLeakDetector {
    private final Map<String, Long> lastCheck = new HashMap<>();
    private final Map<String, Long> growthRate = new HashMap<>();

    public void checkCollectionGrowth(String name, long currentSize) {
        Long previousSize = lastCheck.get(name);
        if (previousSize != null) {
            long growth = currentSize - previousSize;
            if (growth > 0) {
                growthRate.merge(name, growth, Long::sum);
                System.out.printf("%s grew by %d (total growth: %d)%n",
                    name, growth, growthRate.get(name));
                
                if (growthRate.get(name) > 10000) {
                    System.err.println("WARNING: " + name + " may be leaking!");
                }
            }
        }
        lastCheck.put(name, currentSize);
    }

    // Usage example
    public static void main(String[] args) {
        MemoryLeakDetector detector = new MemoryLeakDetector();
        
        // Monitor a cache
        Map<String, Object> cache = new HashMap<>();
        for (int i = 0; i < 1000; i++) {
            cache.put("key" + i, new byte[1024]);
            detector.checkCollectionGrowth("cache", cache.size());
        }
    }
}
```

---

## 5. JVM Flag Inspector

```java
import java.lang.management.ManagementFactory;
import java.lang.management.RuntimeMXBean;
import java.util.List;

public class JVMInspector {
    public static void printJVMInfo() {
        RuntimeMXBean runtimeBean = ManagementFactory.getRuntimeMXBean();
        
        System.out.println("=== JVM Info ===");
        System.out.println("VM Name: " + runtimeBean.getVmName());
        System.out.println("VM Version: " + runtimeBean.getVmVersion());
        System.out.println("VM Vendor: " + runtimeBean.getVmVendor());
        System.out.println("Start Time: " + new Date(runtimeBean.getStartTime()));
        System.out.println("Uptime: " + runtimeBean.getUptime() + "ms");

        System.out.println("\n=== JVM Arguments ===");
        List<String> args = runtimeBean.getInputArguments();
        for (String arg : args) {
            System.out.println(arg);
        }

        System.out.println("\n=== System Properties ===");
        runtimeBean.getSystemProperties().forEach((k, v) -> 
            System.out.println(k + "=" + v));
    }

    public static void printRuntimeInfo() {
        Runtime runtime = Runtime.getRuntime();
        System.out.println("\n=== Runtime Info ===");
        System.out.println("Available processors: " + runtime.availableProcessors());
        System.out.println("Max memory: " + runtime.maxMemory() / 1024 / 1024 + " MB");
        System.out.println("Total memory: " + runtime.totalMemory() / 1024 / 1024 + " MB");
        System.out.println("Free memory: " + runtime.freeMemory() / 1024 / 1024 + " MB");
        System.out.println("Used memory: " + 
            (runtime.totalMemory() - runtime.freeMemory()) / 1024 / 1024 + " MB");
    }
}
```

---

## 6. Custom Health Check Endpoint

```java
import java.lang.management.*;
import java.util.*;

@RestController
public class HealthController {
    
    @GetMapping("/health")
    public Map<String, Object> health() {
        Map<String, Object> health = new HashMap<>();
        
        // Memory
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heap = memoryBean.getHeapMemoryUsage();
        health.put("heapUsedMB", heap.getUsed() / 1024 / 1024);
        health.put("heapMaxMB", heap.getMax() / 1024 / 1024);
        health.put("heapUsagePercent", 
            (double) heap.getUsed() / heap.getMax() * 100);
        
        // GC
        List<GarbageCollectorMXBean> gcBeans = 
            ManagementFactory.getGarbageCollectorMXBeans();
        long totalGcTime = gcBeans.stream()
            .mapToLong(GarbageCollectorMXBean::getCollectionTime)
            .sum();
        health.put("totalGcTimeMs", totalGcTime);
        
        // Threads
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        health.put("threadCount", threadBean.getThreadCount());
        health.put("peakThreadCount", threadBean.getPeakThreadCount());
        
        // Deadlock check
        long[] deadlocked = threadBean.findDeadlockedThreads();
        health.put("deadlocked", deadlocked != null && deadlocked.length > 0);
        
        return health;
    }
}
```

---

## 7. GC Pause Detector

```java
import java.lang.management.*;
import java.util.*;

public class GCPauseDetector {
    private final List<GarbageCollectorMXBean> gcBeans;
    private final Map<String, Long> lastGcTimes = new HashMap<>();
    private final Map<String, Long> lastGcCounts = new HashMap<>();

    public GCPauseDetector() {
        this.gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        // Initialize
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            lastGcTimes.put(gcBean.getName(), gcBean.getCollectionTime());
            lastGcCounts.put(gcBean.getName(), gcBean.getCollectionCount());
        }
    }

    public List<String> checkForLongPauses(long thresholdMs) {
        List<String> warnings = new ArrayList<>();
        
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            String name = gcBean.getName();
            long currentCount = gcBean.getCollectionCount();
            long currentTime = gcBean.getCollectionTime();
            
            long lastCount = lastGcCounts.getOrDefault(name, 0L);
            long lastTime = lastGcTimes.getOrDefault(name, 0L);
            
            if (currentCount > lastCount) {
                long pauseTime = currentTime - lastTime;
                if (pauseTime > thresholdMs) {
                    warnings.add(String.format(
                        "GC '%s' pause: %dms (threshold: %dms)",
                        name, pauseTime, thresholdMs));
                }
            }
            
            lastGcTimes.put(name, currentTime);
            lastGcCounts.put(name, currentCount);
        }
        
        return warnings;
    }
}
```

---

## Key Patterns

| Pattern | Tool/API | Purpose |
|---------|----------|---------|
| GC monitoring | `GarbageCollectorMXBean` | Track GC frequency and pause times |
| Memory monitoring | `MemoryMXBean` | Track heap and non-heap usage |
| Thread monitoring | `ThreadMXBean` | Detect deadlocks and thread contention |
| JVM inspection | `RuntimeMXBean` | Check JVM flags and configuration |
| Native memory | `VM.native_memory` | Track off-heap memory |
| Health checks | Custom endpoints | Expose metrics via HTTP |
