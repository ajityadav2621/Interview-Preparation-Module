# Runnable vs Callable — When Would You Choose Each?

## Overview

Both `Runnable` and `Callable` are functional interfaces used to define tasks that can be executed by threads. However, they have important differences that make them suitable for different scenarios.

## Key Differences

| Feature | `Runnable` | `Callable` |
|---------|-----------|------------|
| **Package** | `java.lang` | `java.util.concurrent` |
| **Return Value** | No return value (`void`) | Returns a value (`V`) |
| **Exceptions** | Cannot throw checked exceptions | Can throw checked exceptions |
| **Method** | `run()` | `call()` |
| **Java Version** | Since Java 1.0 | Since Java 1.5 |
| **Thread Creation** | Can be used with `new Thread()` | Cannot be used with `new Thread()` directly |
| **Future** | No `Future` returned | Returns `Future<V>` |

## Runnable

### Basic Usage

```java
// Runnable — no return value, no checked exceptions
Runnable task = () -> {
    System.out.println("Task is running");
    // Cannot return a value
    // Cannot throw checked exceptions
};

Thread thread = new Thread(task);
thread.start();
```

### Runnable with Thread

```java
public class RunnableExample {
    public static void main(String[] args) {
        Runnable task = () -> {
            String threadName = Thread.currentThread().getName();
            System.out.println("Hello from " + threadName);
        };

        Thread thread = new Thread(task, "MyThread");
        thread.start();
    }
}
```

### Limitations of Runnable

1. **No return value**: You can't get a result from the task
2. **No checked exceptions**: You can't propagate checked exceptions

```java
// This won't compile — Runnable.run() can't throw checked exceptions
Runnable task = () -> {
    Thread.sleep(1000);  // Compilation error!
};
```

## Callable

### Basic Usage

```java
// Callable — returns a value, can throw checked exceptions
Callable<String> task = () -> {
    Thread.sleep(1000);
    return "Task completed";
};

ExecutorService executor = Executors.newSingleThreadExecutor();
Future<String> future = executor.submit(task);

String result = future.get();  // Blocks until task completes
System.out.println(result);    // "Task completed"
executor.shutdown();
```

### Callable with ExecutorService

```java
public class CallableExample {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        Callable<Integer> task = () -> {
            Thread.sleep(1000);
            return 42;
        };

        Future<Integer> future = executor.submit(task);

        // Do other work while task runs...

        Integer result = future.get();  // Blocks until done
        System.out.println("Result: " + result);  // "Result: 42"

        executor.shutdown();
    }
}
```

### Callable with Checked Exceptions

```java
Callable<String> task = () -> {
    // Can throw checked exceptions
    Thread.sleep(1000);
    Files.readAllLines(Paths.get("file.txt"));  // Can throw IOException
    return "Done";
};
```

## When to Use Each

### Use `Runnable` when:

1. **You don't need a return value** — the task is a side-effect operation
2. **You want to use `new Thread()` directly** — Callable can't be used with Thread
3. **You don't need to propagate checked exceptions**
4. **Simple background tasks** — logging, cleanup, notifications

```java
// Good use case for Runnable:
Runnable logger = () -> {
    logger.info("Processing batch " + batchId);
};
executor.submit(logger);
```

### Use `Callable` when:

1. **You need a return value** — the task produces a result
2. **You need to propagate checked exceptions**
3. **You want to use `Future` for async result handling**
4. **You need timeout handling** on the result

```java
// Good use case for Callable:
Callable<List<User>> fetchUsers = () -> {
    return userRepository.findAll();  // Returns a result
};

Future<List<User>> future = executor.submit(fetchUsers);
List<User> users = future.get(5, TimeUnit.SECONDS);  // With timeout
```

## Converting Between Runnable and Callable

### Runnable to Callable

```java
Runnable runnable = () -> System.out.println("Hello");
Callable<Void> callable = Executors.callable(runnable);
```

### Callable to Runnable (discard result)

```java
Callable<String> callable = () -> "result";
Runnable runnable = () -> {
    try {
        callable.call();  // Execute but ignore result
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
};
```

## Future and CompletableFuture

### With Callable + Future

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

List<Future<Integer>> futures = new ArrayList<>();
for (int i = 0; i < 10; i++) {
    final int taskId = i;
    Callable<Integer> task = () -> {
        Thread.sleep(1000);
        return taskId * taskId;
    };
    futures.add(executor.submit(task));
}

// Collect results
for (Future<Integer> future : futures) {
    System.out.println(future.get());  // Blocks
}
executor.shutdown();
```

### With CompletableFuture (Java 8+)

```java
CompletableFuture<Integer> future = CompletableFuture
    .supplyAsync(() -> {
        try { Thread.sleep(1000); } catch (InterruptedException e) {}
        return 42;
    });

future.thenAccept(result -> System.out.println("Result: " + result));
```

## Code Example: Choosing Between Runnable and Callable

```java
public class TaskComparison {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        // Runnable — fire and forget
        Runnable notificationTask = () -> {
            sendEmail("user@example.com", "Welcome!");
        };
        executor.submit(notificationTask);

        // Callable — need the result
        Callable<User> userFetchTask = () -> {
            return database.findUserById(123);
        };
        Future<User> userFuture = executor.submit(userFetchTask);
        User user = userFuture.get();  // Get the result

        executor.shutdown();
    }
}
```

## Key Takeaway

> Use `Runnable` for simple, fire-and-forget tasks that don't need a return value. Use `Callable` when you need a return value, need to handle checked exceptions, or want to use `Future` for async result management. In modern Java, `CompletableFuture` (which uses `Supplier`, similar to `Callable`) is often preferred for complex async workflows.
