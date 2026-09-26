# CompletableFuture

## What Problem Does It Solve?

### Before CompletableFuture
```java
// Complex async operations required manual coordination
Future<String> future1 = executor.submit(task1);
Future<String> future2 = executor.submit(task2);

// Blocking and manual composition
while (!future1.isDone() || !future2.isDone()) {
    Thread.sleep(100);
}
String result1 = future1.get();
String result2 = future2.get();
```

### With CompletableFuture
```java
// Simple composition
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> task1())
    .thenCompose(res1 -> CompletableFuture
        .supplyAsync(() -> task2())
        .thenApply(res2 -> res1 + res2));

// Non-blocking
result.thenAccept(finalResult -> {
    System.out.println("Final: " + finalResult);
});
```

## Key Features

### 1. **Composition**
```java
// Chain operations
CompletableFuture.supplyAsync(() -> fetchData())
    .thenApply(data -> process(data))
    .thenAccept(result -> save(result));

// Parallel execution
CompletableFuture.allOf(task1, task2, task3)
    .thenRun(() -> System.out.println("All done"));
```

### 2. **Error Handling**
```java
CompletableFuture.supplyAsync(() -> riskyOperation())
    .exceptionally(ex -> {
        log.error("Operation failed", ex);
        return "default";
    });
```

### 3. **Timeout Handling**
```java
CompletableFuture.supplyAsync(() -> longOperation())
    .orTimeout(5, TimeUnit.SECONDS);
```

## When to Use
- **Parallel API calls** - combine multiple services
- **Data transformation pipelines** - sequential operations
- **Timeout scenarios** - fail fast
- **Complex async workflows** - multiple dependent tasks

## Interview Tip
Explain that CompletableFuture makes async code more readable and maintainable. Mention that it's part of Java 8 and became much better in Java 9+ with new methods like orTimeout.