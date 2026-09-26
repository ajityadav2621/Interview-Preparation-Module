# Exception Inside Thread

## What Happens?

### 1. **Uncaught Exception**
```java
Thread thread = new Thread(() -> {
    throw new RuntimeException("Oops!");
});
thread.start(); // Exception propagates to thread's uncaught exception handler
```

### 2. **Default Behavior**
- Thread terminates
- Exception printed to console
- Main thread continues (unless it's the main thread)

### 3. **Custom Exception Handler**
```java
Thread thread = new Thread(() -> {
    throw new RuntimeException("Error in task");
});

thread.setUncaughtExceptionHandler((t, ex) -> {
    System.err.println("Thread " + t.getName() + " failed: " + ex.getMessage());
    // Log, alert, cleanup
});

thread.start();
```

## With ExecutorService

### 1. **Using submit()**
```java
Future<?> future = executor.submit(() -> {
    throw new RuntimeException("Task failed");
});

try {
    future.get(); // Exception thrown here
} catch (ExecutionException e) {
    // Handle exception
    Throwable cause = e.getCause();
}
```

### 2. **Using execute()**
```java
executor.execute(() -> {
    throw new RuntimeException("Task failed");
});
// Exception handled by thread's uncaught exception handler
```

## Best Practices
- Always handle exceptions in tasks
- Use custom uncaught exception handlers
- For Callable tasks, handle ExecutionException
- Implement proper logging and monitoring

## Interview Tip
Explain that exceptions in threads don't propagate to the calling thread. Mention the difference between execute() and submit() in ExecutorService.