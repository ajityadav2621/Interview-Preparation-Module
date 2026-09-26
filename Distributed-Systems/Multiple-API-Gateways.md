# Designing Multiple API Gateway Instances

## Architecture

### 1. **Load Balancer Distribution**
```
Client → Load Balancer → Gateway 1, Gateway 2, Gateway 3
                      ↓
                Each gateway handles subset of traffic
```

### 2. **Stateless Gateway Design**
```java
// Gateway should be stateless
@RestController
@RequestMapping("/api")
public class GatewayController {
    
    private final RouteLocator routeLocator;
    private final LoadBalancer loadBalancer;
    
    @GetMapping("/{service}/**")
    public ResponseEntity<?> proxyRequest(@PathVariable String service,
                                         HttpServletRequest request) {
        // Route to appropriate service
        String targetUrl = loadBalancer.chooseService(service);
        return RestTemplate.forward(request, targetUrl);
    }
}
```

### 3. **Configuration Management**
```yaml
# Dynamic routing configuration
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/users/**
```

## Benefits
- High availability
- Horizontal scaling
- Load distribution
- Zero downtime deployments

## Interview Tip
Explain that API gateways should be stateless to enable horizontal scaling. Use service discovery for dynamic routing.