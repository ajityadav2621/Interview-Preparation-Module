# Detecting Memory Leaks with Observability Tools

## JVM Monitoring
```bash
# Monitor heap usage
jstat -gc <pid>

# Look for:
# Old generation increasing
# Full GC frequency
# Memory not being reclaimed
```

## Heap Dump Analysis
```bash
# Generate heap dump
jmap -dump:live,format=b,file=heap.hprof <pid>

# Analyze with Eclipse MAT
# Look for:
# Dominator tree (who holds memory)
# Leak suspects
# Top consumers
```

## Application Metrics
```java
private final Gauge heapUsedGauge;

public MemoryMetrics(MeterRegistry registry) {
    this.heapUsedGauge = Gauge.builder("jvm.heap.used")
        .register(registry);
}
```

## Interview Tip
Explain that memory leak detection requires correlating JVM metrics with heap dump analysis. Always check for increasing old generation usage after GC.