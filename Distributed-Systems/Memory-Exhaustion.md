# Microservice Runs Out of Memory

## Causes
- Memory leaks
- Large object allocations
- Unbounded caches
- Heap size too small
- Session accumulation

## Symptoms
- OutOfMemoryError
- Frequent Full GC
- Slow response times
- Container restarts

## Solutions

### 1. **Heap Dump Analysis**
```bash
# Generate heap dump
jmap -dump:live,format=b,file=heap.hprof <pid>

# Analyze with Eclipse MAT
# Look for:
# - Dominator tree (who holds memory)
# - Leak suspects
# - Top consumers
```

### 2. **Increase Heap Size**
```java
# JVM flags
-Xms2g -Xmx2g -XX:MaxMetaspaceSize=256m
```

### 3. **Fix Memory Issues**
```java
// Avoid memory leaks
public class UserService {
    // Bad: static collection grows unbounded
    private static List<User> cache = new ArrayList<>();
    
    // Good: bounded cache
    private final Cache<String, User> cache = Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(30, TimeUnit.MINUTES)
        .build();
}
```

### 4. **Kubernetes Configuration**
```yaml
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "1Gi"
```

## Interview Tip
Explain that memory issues in microservices often require heap dump analysis. Always check for memory leaks before increasing heap size.