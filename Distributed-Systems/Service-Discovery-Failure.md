# Service Discovery Failure - What Happens?

## Impact
- Cannot locate downstream services
- Requests fail with "service not found"
- Cascading failures
- Partial system outage

## What Should Happen

### 1. **Caching Service Information**
```java
// Cache service locations
@Cacheable("services")
public String getServiceUrl(String serviceName) {
    return discoveryClient.getInstances(serviceName)
        .stream()
        .map(ServiceInstance::getUri)
        .findFirst()
        .orElseThrow(() -> new ServiceNotFoundException());
}
```

### 2. **Retry with Backoff**
```java
@Retryable(value = {Exception.class},
           maxAttempts = 3,
           backoff = @Backoff(delay = 2000))
public String callService(String serviceName) {
    String url = getServiceUrl(serviceName);
    return RestTemplate.getForObject(url, String.class);
}
```

### 3. **Fallback Response**
```java
@CircuitBreaker(name = "service", fallbackMethod = "fallback")
public Data callService() {
    return service.call();
}

public Data fallback(Exception ex) {
    // Return cached data or default
    return cache.getOrDefault("default", Data.getDefault());
}
```

## Interview Tip
Explain that service discovery is critical in dynamic environments. Always implement caching and fallbacks to handle temporary failures.