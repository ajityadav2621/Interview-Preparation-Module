# JVM Metrics to Monitor

## Critical Metrics

### 1. **Heap Usage**
- Young generation usage
- Old generation usage
- Metaspace usage
- Full GC frequency

### 2. **GC Performance**
- GC pause times
- GC frequency
- Allocation rate
- Promotion rate

### 3. **Thread Metrics**
- Thread count
- Thread states (RUNNABLE, BLOCKED, WAITING)
- Deadlock detection

### 4. **System Metrics**
- CPU usage
- Memory usage
- Disk I/O
- Network traffic

## Monitoring Tools
```bash
# JMX metrics
jstat -gc <pid>
jstack <pid>

# Spring Boot Actuator
GET /actuator/metrics
GET /actuator/prometheus
```

## Interview Tip
Explain that JVM monitoring helps identify memory issues, thread contention, and performance bottlenecks before they cause outages.