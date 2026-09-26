# Thread Pools in Java

## Types of Thread Pools

### 1. **Fixed Thread Pool**
```java
ExecutorService fixedPool = Executors.newFixedThreadPool(10);
```
- **Use Case**: Controlled concurrency, limited resources
- **Characteristics**: Fixed number of threads, unbounded queue
- **When to Use**: CPU-bound tasks, known workload

### 2. **Cached Thread Pool**
```java
ExecutorService cachedPool = Executors.newCachedThreadPool();
```
- **Use Case**: Many short-lived tasks
- **Characteristics**: Creates new threads as needed, reuses idle threads
- **When to Use**: I/O-bound tasks, quick operations

### 3. **Scheduled Thread Pool**
```java
ScheduledExecutorService scheduledPool = Executors.newScheduledThreadPool(5);
```
- **Use Case**: Periodic tasks, delays
- **Characteristics**: Schedule tasks with delay or periodically
- **When to Use**: Cron jobs, periodic checks

### 4. **Single Thread Executor**
```java
ExecutorService singleThread = Executors.newSingleThreadExecutor();
```
- **Use Case**: Sequential processing
- **Characteristics**: Single thread, guaranteed order
- **When to Use**: Order-sensitive operations

## Choosing the Right Pool

### Factors to Consider

#### **Task Type**
- **CPU-bound**: Fixed pool sized to CPU cores
- **I/O-bound**: Larger pool, more threads than cores
- **Mixed**: Consider work queue and thread count

#### **Queue Strategy**
- **Unbounded queue**: Risk of OOM under heavy load
- **Bounded queue**: Prevents overload, may reject tasks
- **Direct handoff**: No queue, immediate thread allocation

#### **Rejection Policy**
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10, 20, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100),
    new ThreadPoolExecutor.AbortPolicy()
);
```

### Custom Pool Configuration
```java
// For I/O-bound tasks
int ioPoolSize = Runtime.getRuntime().availableProcessors() * 2;
ExecutorService ioPool = Executors.newFixedThreadPool(ioPoolSize);

// For CPU-bound tasks
int cpuPoolSize = Runtime.getRuntime().availableProcessors();
ExecutorService cpuPool = Executors.newFixedThreadPool(cpuPoolSize);
```

## Interview Tip
Explain that thread pool sizing depends on task characteristics. Mention that using too few threads causes underutilization, while too many causes contention.