# Preventing Tenant Resource Consumption

## Multi-Tenancy Challenges
- One tenant can consume all resources
- Performance isolation
- Fair resource allocation

## Solutions

### 1. **Resource Quotas**
```yaml
# Per-tenant resource limits
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
```

### 2. **Rate Limiting by Tenant**
```java
// Per-tenant rate limiting
Map<String, Bucket> tenantBuckets = new ConcurrentHashMap<>();

public boolean allowRequest(String tenantId) {
    Bucket bucket = tenantBuckets.computeIfAbsent(tenantId, 
        k -> Bucket.builder()
            .addLimit(Bandwidth.classic(1000, Duration.ofMinutes(1))
            .build());
    
    return bucket.tryConsume(1);
}
```

### 3. **Priority Queues**
```java
// Different queues for different tenants
@Qualifier("highPriority")
ExecutorService highPriorityExecutor;

@Qualifier("lowPriority") 
ExecutorService lowPriorityExecutor;
```

### 4. **Monitoring and Alerting**
```java
// Track per-tenant usage
@Metric
private Meter tenantRequests;

public void recordTenantUsage(String tenantId, long count) {
    tenantRequests.tag("tenant", tenantId).increment(count);
}
```

## Interview Tip
Explain that multi-tenancy requires explicit resource isolation. Use quotas, rate limiting, and monitoring to ensure fair usage.