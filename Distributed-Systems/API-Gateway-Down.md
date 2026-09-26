# API Gateway Goes Down - What Happens?

## Impact
- All client requests fail
- Single point of failure
- No routing, authentication, or rate limiting

## What Should Happen

### 1. **Multiple Gateway Instances**
```yaml
# Deploy multiple gateway instances
replicas: 3

# Load balancer distributes traffic
service:
  type: LoadBalancer
  port: 8080
```

### 2. **Client-Side Fallback**
```java
// Circuit breaker for gateway
@CircuitBreaker(name = "gateway", fallbackMethod = "gatewayFallback")
public Response callApi(String endpoint) {
    return gatewayClient.call(endpoint);
}

public Response gatewayFallback(String endpoint, Exception ex) {
    // Return cached response or error
    return Response.status(503).entity("Service temporarily unavailable").build();
}
```

### 3. **Health Checks**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
```

## Interview Tip
Explain that API gateways should be deployed with high availability. Use load balancers and health checks to ensure redundancy.