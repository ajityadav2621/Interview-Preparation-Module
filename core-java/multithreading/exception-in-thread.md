# What Happens if an Exception Is Thrown Inside a Thread?

## The Problem

When an exception is thrown inside a thread, the behavior depends on **how the thread was created** and **how the exception is handled**.

## Default Behavior: Thread Dies Silently

### With `Thread` (Runnable)

```java
Thread thread = new Thread(() -> {
    throw new RuntimeException("Something went wrong!");
});
thread.start();

// The thread dies, but:
// 1. The exception is printed to stderr (via ThreadGroup's default handler)
// 2. No other thread is notified
// 3. No way to catch the exception from the calling thread
// 4. The thread is gone — no recovery
```

**Output**:
```
Exception in thread "Thread-0" java.lang.RuntimeException: Something went wrong!
    at MyClass.lambda$main$0(MyClass.java:10)
```

### With `ExecutorService`

```java
ExecutorService executor = Executors.newFixedThreadPool(2);
Future<?> future = executor.submit(() -> {
    throw new RuntimeException("Task failed!");
});

// The exception is captured in the Future
try {
    future.get();  // Throws ExecutionException wrapping the RuntimeException
} catch (ExecutionException e) {
    Throwable cause = e.getCause();
    System.out.println("Task failed: " + cause.getMessage());
}
```

## Why This Matters

### 1. Silent Failures

```java
// This is dangerous — the thread dies silently
for (int i = 0; i < 100; i++) {
    new Thread(() -> {
        processOrder(orderId);  // If this throws, the thread dies silently
    }).start();
}
// No way to know if any orders failed!
```

### 2. Resource Leaks

```java
Thread thread = new Thread(() -> {
    Connection conn = dataSource.getConnection();
    try {
        process(conn);
    } finally {
        conn.close();  // If exception is thrown before this, connection leaks!
    }
});
thread.start();
// If the thread dies, the connection is never closed
```

### 3. No Recovery

```java
// Thread dies — no way to restart or retry
Thread worker = new Thread(() -> {
    while (true) {
        try {
            processTask();
        } catch (Exception e) {
            // If we don't catch here, the thread dies
            throw new RuntimeException(e);
        }
    }
});
worker.start();
```

## Solutions

### 1. UncaughtExceptionHandler

You can set a default handler for uncaught exceptions:

```java
Thread thread = new Thread(() -> {
    throw new RuntimeException("Task failed!");
});

thread.setUncaughtExceptionHandler((t, e) -> {
    System.err.println("Thread " + t.getName() + " failed: " + e.getMessage());
    // Log, alert, restart, etc.
});

thread.start();
```

### 2. Default UncaughtExceptionHandler

Set a default for all threads:

```java
Thread.setDefaultUncaughtExceptionHandler((t, e) -> {
    System.err.println("Uncaught exception in thread " + t.getName());
    e.printStackTrace();
    // Send alert, log to monitoring system, etc.
});
```

### 3. Try-Catch Inside the Thread

```java
Thread thread = new Thread(() -> {
    try {
        processTask();
    } catch (Exception e) {
        System.err.println("Task failed: " + e.getMessage());
        // Handle the exception — log, retry, etc.
    }
});
thread.start();
```

### 4. Use ExecutorService (Recommended)

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

for (int i = 0; i < 100; i++) {
    final int taskId = i;
    Future<?> future = executor.submit(() -> {
        processTask(taskId);
    });

    // Check for exceptions
    try {
        future.get();  // Will throw ExecutionException if task failed
    } catch (ExecutionException e) {
        Throwable cause = e.getCause();
        System.err.println("Task " + taskId + " failed: " + cause.getMessage());
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}
```

### 5. CompletableFuture Exception Handling

```java
CompletableFuture<Void> future = CompletableFuture
    .runAsync(() -> processTask())
    .exceptionally(ex -> {
        System.err.println("Task failed: " + ex.getMessage());
        return null;
    });
```

## Thread Groups and Exception Propagation

### ThreadGroup Default Handler

```java
// ThreadGroup has a default uncaughtException handler
// It prints the stack trace to stderr
// You can override it:
ThreadGroup group = new ThreadGroup("MyGroup") {
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.err.println("Custom handler: " + e.getMessage());
    }
};
Thread thread = new Thread(group, () -> {
    throw new RuntimeException("Failed!");
});
thread.start();
```

## Best Practices

### 1. Always Handle Exceptions in Threads

```java
// BAD:
executor.submit(() -> processTask());

// GOOD:
Future<?> future = executor.submit(() -> processTask());
try {
    future.get();
} catch (ExecutionException e) {
    handleError(e.getCause());
}
```

### 2. Use CompletableFuture for Better Error Handling

```java
CompletableFuture.supplyAsync(() -> processTask())
    .thenAccept(result -> handleSuccess(result))
    .exceptionally(ex -> {
        handleError(ex);
        return null;
    });
```

### 3. Set a Default UncaughtExceptionHandler

```java
// In your application startup:
Thread.setDefaultUncaughtExceptionHandler((t, e) -> {
    logger.error("Uncaught exception in thread: " + t.getName(), e);
    // Send alert to monitoring system
    alertService.sendAlert("Thread " + t.getName() + " crashed: " + e.getMessage());
});
```

### 4. Use Thread Pools with Proper Error Handling

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4, 4, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>()
) {
    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        super.afterExecute(r, t);
        if (t != null) {
            // Log the exception
            logger.error("Task failed", t);
        } else if (r instanceof Future<?>) {
            try {
                ((Future<?>) r).get();
            } catch (ExecutionException e) {
                logger.error("Task failed", e.getCause());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
};
```

## Key Takeaway

> When an exception is thrown inside a thread, the thread dies and the exception is either printed to stderr (for `Thread`) or captured in the `Future` (for `ExecutorService`). **Always handle exceptions in threads** — use `Future.get()` to check for failures, set `UncaughtExceptionHandler` for safety, and prefer `CompletableFuture` or `ExecutorService` over raw `Thread` for better error handling and recovery.
