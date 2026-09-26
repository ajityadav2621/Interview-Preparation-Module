# Thread Contention Issues in Production

## What is Thread Contention?

Thread contention occurs when multiple threads compete for the same resource (lock, database connection, I/O), leading to performance degradation.

## How to Analyze

### 1. **Thread Dump Analysis**
```bash
# Get thread dump
jstack <pid> > thread_dump.txt

# Look for BLOCKED state
grep "BLOCKED" thread_dump.txt

# Check lock ownership
grep -A 5 "BLOCKED" thread_dump.txt
```

### 2. **JVM Monitoring**
```bash
# Monitor thread states
jstat -thread <pid>

# Monitor GC activity
jstat -gc <pid>
```

### 3. **Application Metrics**
- Thread pool queue size
- Database connection pool usage
- Lock wait times
- CPU vs I/O distribution

## Common Causes

### 1. **Database Connection Pool Exhaustion**
```java
// Symptoms
- Threads waiting for connections
- High latency
- Connection timeout errors

// Solution
- Increase pool size
- Optimize queries
- Use read replicas
```

### 2. **Lock Contention**
```java
// Heavy synchronization on shared resources
private final Lock lock = new ReentrantLock();

// Too many threads competing
for (int i = 0; i < 1000; i++) {
    executor.submit(() -> {
        lock.lock();
        try {
            // Critical section
        } finally {
            lock.unlock();
        }
    });
}
```

### 3. **I/O Bottlenecks**
- Slow database queries
- External API calls
- File system operations

## Fixing Contention Issues

### 1. **Reduce Critical Section Size**
```java
// BAD - large critical section
lock.lock();
try {
    // Many operations
    database.save();
    externalApi.call();
} finally {
    lock.unlock();
}

// GOOD - minimal critical section
lock.lock();
try {
    // Quick operation
    updateLocalState();
} finally {
    lock.unlock();
}

// Do slow operations outside lock
database.save();
externalApi.call();
```

### 2. **Use Concurrent Data Structures**
```java
// Instead of synchronized list
List<String> list = new CopyOnWriteArrayList<>();

// Instead of synchronized map
Map<String, Object> map = new ConcurrentHashMap<>();
```

### 3. **Tune Thread Pools**
```java
// For I/O-bound tasks
int ioThreads = Runtime.getRuntime().availableProcessors() * 2;
ExecutorService ioExecutor = Executors.newFixedThreadPool(ioThreads);

// For CPU-bound tasks
int cpuThreads = Runtime.getRuntime().availableProcessors();
ExecutorService cpuExecutor = Executors.newFixedThreadPool(cpuThreads);
```

## Interview Tip
Explain that contention analysis requires looking at both application metrics and system resources. Mention that reducing lock scope is often the most effective fix.