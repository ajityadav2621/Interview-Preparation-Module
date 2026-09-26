# Bulkhead Pattern

## What is Bulkhead?
Bulkhead pattern isolates resources to prevent one failure from affecting the entire system. Named after ship bulkheads that contain flooding.

## Implementation

### 1. **Thread Pool Isolation**
```java
// Separate thread pools for different services
ExecutorService criticalService = Executors.newFixedThreadPool(10);
ExecutorService nonCriticalService = Executors.newFixedThreadPool(5);

// Critical service requests use dedicated threads
CompletableFuture.runAsync(() -> criticalService.call(), criticalExecutor);

// Non-critical service uses separate pool
CompletableFuture.runAsync(() -> nonCriticalService.call(), nonCriticalExecutor);
```

### 2. **Resilience4j Bulkhead**
```java
Bulkhead bulkhead = Bulkhead.of(10, TimeUnit.SECONDS);

Supplier<String> decorated = Bulkhead
    .decorateSupplier(bulkhead, () -> externalApi.call());
```

### 3. **Semaphore-Based**
```java
Semaphore semaphore = new Semaphore(10);

try {
    semaphore.acquire();
    // Critical section
    processRequest();
} finally {
    semaphore.release();
}
```

## Benefits
- Prevents resource exhaustion
- Isolates failures
- Maintains partial functionality
- Predictable performance

## Interview Tip
Explain that bulkheads are like firewalls for resources. They ensure one service's failure doesn't consume all available threads or connections.