# An External API Takes 20 Seconds to Respond. How Would You Protect Your Service?

## The Problem

An external API that takes 20 seconds to respond can cause:

1. **Thread exhaustion** — request threads are blocked waiting
2. **Cascading failures** — slow responses propagate through the system
3. **Poor user experience** — users wait for the slow external call
4. **Resource waste** — connections and memory held for too long

## Solutions

### 1. Timeout Configuration

Set aggressive timeouts to fail fast:

```java
// Using RestTemplate
@Bean
public RestTemplate restTemplate() {
    HttpComponentsClientHttpRequestFactory factory = 
        new HttpComponentsClientHttpRequestFactory();
    factory.setConnectTimeout(2000);  // 2 seconds to connect
    factory.setReadTimeout(5000);     // 5 seconds to read response
    return new RestTemplate(factory);
}

// Using WebClient (reactive)
@Bean
public WebClient webClient() {
    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(
            HttpClient.create()
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000)
                .responseTimeout(Duration.ofSeconds(5))
        ))
        .build();
}
```

### 2. Circuit Breaker

Use a circuit breaker to fail fast when the external service is slow:

```java
@Service
public class ExternalApiService {
    
    private final CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("external-api");
    
    public ApiResponse callExternalApi(Request request) {
        return circuitBreaker.executeSupplier(() -> {
            return externalApiClient.call(request);
        });
    }
}
```

```yaml
# Resilience4j circuit breaker configuration
resilience4j:
  circuitbreaker:
    instances:
      external-api:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        sliding-window-size: 10
        minimum-number-of-calls: 5
```

### 3. Bulkhead Pattern

Limit concurrent calls to the external service:

```java
@Service
public class ExternalApiService {
    
    private final ThreadPoolBulkhead bulkhead = 
        ThreadPoolBulkhead.of("external-api",
            ThreadPoolBulkheadConfig.custom()
                .coreThreadPoolSize(5)
                .maxThreadPoolSize(10)
                .queueCapacity(20)
                .build());
    
    public CompletableFuture<ApiResponse> callExternalApi(Request request) {
        return bulkhead.executeSupplier(() -> {
            return externalApiClient.call(request);
        });
    }
}
```

### 4. Asynchronous Processing with Timeout

Move the external call to a background thread with a timeout:

```java
@Service
public class OrderService {
    
    @Autowired
    private ExecutorService executor;
    
    public OrderResult processOrder(OrderRequest request) {
        // Quick response — process external call in background
        CompletableFuture.supplyAsync(() -> {
            return externalApiClient.call(request);
        }, executor)
        .orTimeout(5, TimeUnit.SECONDS)  // Timeout after 5 seconds
        .handle((result, ex) -> {
            if (ex != null) {
                // Handle timeout or failure
                return fallbackResult(request);
            }
            return processResult(result);
        });
        
        return OrderResult.accepted("Order is being processed");
    }
}
```

### 5. Caching

Cache responses to avoid repeated slow calls:

```java
@Service
public class ExternalApiService {
    
    @Cacheable(value = "external-api", key = "#request.id", 
               unless = "#result == null")
    @CacheEvict(value = "external-api", key = "#request.id", 
                condition = "#result != null")
    public ApiResponse callExternalApi(Request request) {
        return externalApiClient.call(request);
    }
}
```

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 300000  # 5 minutes
```

### 6. Retry with Exponential Backoff

Retry failed calls with increasing delays:

```java
@Retryable(
    value = {TimeoutException.class, ConnectException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000, multiplier = 2)
)
public ApiResponse callExternalApi(Request request) {
    return externalApiClient.call(request);
}

@Recover
public ApiResponse fallback(TimeoutException ex, Request request) {
    return ApiResponse.timeout("External API timed out");
}
```

### 7. Rate Limiting

Protect the external service from being overwhelmed:

```java
@Service
public class ExternalApiService {
    
    private final RateLimiter rateLimiter = RateLimiter.of("external-api",
        RateLimiterConfig.custom()
            .limitForPeriod(10)  // 10 requests per period
            .limitRefreshPeriod(Duration.ofSeconds(1))  // 1 second
            .timeoutDuration(Duration.ofSeconds(5))
            .build());
    
    public ApiResponse callExternalApi(Request request) {
        return rateLimiter.executeSupplier(() -> {
            return externalApiClient.call(request);
        });
    }
}
```

### 8. Fallback Responses

Provide fallback responses when the external service is slow:

```java
@Service
public class ProductService {
    
    public Product getProduct(String productId) {
        try {
            return externalApiClient.getProduct(productId);
        } catch (Exception e) {
            // Return cached or default data
            return cache.get("product:" + productId)
                .orElse(getDefaultProduct(productId));
        }
    }
}
```

### 9. Request Queueing

Queue requests when the external service is slow:

```java
@Service
public class ExternalApiService {
    
    private final BlockingQueue<Request> requestQueue = new LinkedBlockingQueue<>(100);
    private final ExecutorService executor = Executors.newFixedThreadPool(5);
    
    @PostConstruct
    public void startProcessing() {
        for (int i = 0; i < 5; i++) {
            executor.submit(this::processQueue);
        }
    }
    
    public CompletableFuture<ApiResponse> callExternalApi(Request request) {
        CompletableFuture<ApiResponse> future = new CompletableFuture<>();
        requestQueue.offer(new QueuedRequest(request, future));
        return future;
    }
    
    private void processQueue() {
        while (true) {
            try {
                QueuedRequest qr = requestQueue.take();
                ApiResponse response = externalApiClient.call(qr.request);
                qr.future.complete(response);
            } catch (Exception e) {
                // Handle error
            }
        }
    }
}
```

## Complete Protection Strategy

```java
@Service
public class ProtectedExternalApiService {
    
    private final CircuitBreaker circuitBreaker;
    private final ThreadPoolBulkhead bulkhead;
    private final RateLimiter rateLimiter;
    private final Cache<String, ApiResponse> cache;
    
    public ApiResponse callExternalApi(Request request) {
        // 1. Check cache first
        ApiResponse cached = cache.getIfPresent(request.getId());
        if (cached != null) {
            return cached;
        }
        
        // 2. Apply rate limiting
        return rateLimiter.executeSupplier(() -> {
            // 3. Apply circuit breaker and bulkhead
            return circuitBreaker.executeSupplier(() -> {
                return bulkhead.executeSupplier(() -> {
                    // 4. Call with timeout
                    return externalApiClient.call(request);
                }).toCompletableFuture().join();
            });
        });
    }
}
```

## Monitoring

```java
@Component
public class ExternalApiMonitor {
    
    @Scheduled(fixedRate = 30000)
    public void logMetrics() {
        // Log circuit breaker state
        // Log bulkhead queue depth
        // Log rate limiter metrics
        // Log cache hit ratio
        // Log external API latency
    }
}
```

## Key Takeaway

> To protect your service from a slow external API, use a **layered defense**: **timeouts** to fail fast, **circuit breakers** to prevent cascading failures, **bulkheads** to limit concurrent calls, **rate limiting** to protect the external service, **caching** to avoid repeated calls, and **fallback responses** to provide graceful degradation. Monitor all these metrics to detect issues before they impact users.
