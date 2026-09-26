# JVM Garbage Collection Metrics to Monitor

## Key Metrics

### 1. **GC Frequency and Pause Times**
```bash
# Monitor GC activity
jstat -gc <pid> 1000

# Key columns:
# S0C, S1C: Survivor space capacity
# S0U, S1U: Survivor space used
# EC, EU: Eden space capacity/used
# OC, OU: Old generation capacity/used
# YGC: Young GC count
# YGCT: Young GC time
# FGC: Full GC count
# FGCT: Full GC time
```

### 2. **Heap Usage**
- Young generation usage
- Old generation usage
- Metaspace usage
- Overall heap utilization

### 3. **Allocation Rates**
- Object creation rate
- Promotion rate to old generation
- GC overhead percentage

## What to Monitor

### Critical Metrics
- **Full GC frequency** - should be low and stable
- **GC pause times** - should be sub-second for most applications
- **Heap usage after GC** - indicates memory leaks
- **Allocation rate** - helps size the heap

### Warning Signs
- Increasing Full GC frequency
- Long GC pauses (> 1 second)
- Heap usage not decreasing after GC
- High allocation rate leading to frequent GC

## Production Monitoring

### Tools
- **JMX**: Real-time metrics via JConsole/MC4J
- **Prometheus**: Scrapes JMX metrics
- **Java Flight Recorder**: Detailed GC analysis
- **APM Tools**: Dynatrace, AppDynamics

### JVM Flags for Monitoring
```java
-XX:+UseG1GC
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xlog:gc*:file=gc.log
```

## Interview Tip
Explain that GC monitoring helps identify memory issues before they cause outages. Mention that different GC algorithms have different characteristics (G1 for balanced, ZGC for low latency).