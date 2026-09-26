# Full GC and Its Impact on API Latency

## What is Full GC?
Full GC is a garbage collection process that cleans both the Young and Old generations of the heap. It's also known as "Major GC."

## When Does Full GC Happen?
- When Old Generation is full
- When explicitly requested (System.gc())
- When insufficient space for new objects
- During concurrent GC pauses

## Why Full GC Affects API Latency

### 1. **Stop-The-World (STW) Pause**
- All application threads are suspended
- No request processing during GC
- Can last from milliseconds to seconds

### 2. **Impact on Response Time**
```java
// Normal request processing
long start = System.currentTimeMillis();
processRequest();
long duration = System.currentTimeMillis() - start;
// Normal: 50-100ms

// During Full GC
// All threads paused for 2-5 seconds
// Response time becomes 2000-5000ms
```

### 3. ** cascading Effects**
- Thread pools exhausted
- Connection timeouts
- Increased load on remaining instances
- Circuit breakers opening

## Monitoring Full GC

### JVM Metrics
```bash
# Monitor GC activity
jstat -gc <pid> 1000

# Look for:
# YGC (Young GC frequency)
# YGCT (Young GC time)
# FGC (Full GC frequency)  
# FGCT (Full GC time)
```

### Key Indicators
- High FGCT (Full GC time)
- Increasing FGC count
- High heap usage after GC
- STW pauses in logs

## Prevention Strategies
- Tune heap sizes appropriately
- Choose the right GC algorithm (G1, ZGC, Shenandoah)
- Monitor memory usage patterns
- Set appropriate JVM flags

## Interview Tip
Explain that Full GC is a major source of latency spikes in production. Mention that modern GC algorithms like ZGC aim to reduce STW pauses to sub-millisecond levels.