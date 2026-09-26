# Runnable vs Callable

## Core Differences

| Feature | Runnable | Callable |
|---------|----------|----------|
| **Return Value** | No (void) | Yes (generic type) |
| **Exception** | Cannot throw checked exceptions | Can throw checked exceptions |
| **Submission** | execute() | submit() |
| **Future Access** | No | Returns Future<T> |
| **Package** | java.lang | java.util.concurrent |

## When to Use Each

### Use Runnable When:
- **No return value needed** - fire-and-forget tasks
- **Simple operations** - no complex computation
- **Exception handling** - handled internally
- **Performance** - less overhead than Callable

```java
// Good for logging, notifications
Runnable logTask = () -> {
    log.info("User logged in: {}", userId);
};
executor.execute(logTask);
```

### Use Callable When:
- **Need return value** - computation results
- **Checked exceptions** - may throw IOException, etc.
- **Future operations** - need to wait for results
- **Parallel computations** - combine multiple results

```java
// Good for API calls, database queries
Callable<String> apiCall = () -> {
    return httpClient.get("https://api.example.com/data");
};

Future<String> future = executor.submit(apiCall);
String result = future.get(); // Blocks until complete
```

## Practical Examples

### Runnable Example
```java
class PrintTask implements Runnable {
    private final String message;
    
    PrintTask(String message) {
        this.message = message;
    }
    
    @Override
    public void run() {
        System.out.println(message);
    }
}

executor.execute(new PrintTask("Hello"));
```

### Callable Example
```java
class SumTask implements Callable<Integer> {
    private final int a, b;
    
    SumTask(int a, int b) {
        this.a = a;
        this.b = b;
    }
    
    @Override
    public Integer call() throws Exception {
        return a + b;
    }
}

Future<Integer> future = executor.submit(new SumTask(5, 10));
Integer result = future.get();
```

## Interview Tip
Explain that Callable is preferred when you need results from asynchronous operations. Mention that both are functional interfaces and can be used with lambda expressions.