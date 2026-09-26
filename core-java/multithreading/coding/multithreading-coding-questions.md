# Multithreading Coding Questions

## 1. Thread-Safe Counter

```java
// Using synchronized
public class SynchronizedCounter {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }

    public synchronized void decrement() {
        count--;
    }

    public synchronized int getCount() {
        return count;
    }
}

// Using AtomicInteger (lock-free)
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();
    }

    public void decrement() {
        count.decrementAndGet();
    }

    public int getCount() {
        return count.get();
    }
}

// Using ReentrantLock
import java.util.concurrent.locks.ReentrantLock;

public class LockedCounter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();

    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }

    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 2. Producer-Consumer with wait/notify

```java
import java.util.LinkedList;
import java.util.Queue;

public class ProducerConsumer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 5;

    public void produce() throws InterruptedException {
        int value = 0;
        while (true) {
            synchronized (this) {
                while (queue.size() == capacity) {
                    wait();  // Wait for consumer to consume
                }
                queue.add(value++);
                System.out.println("Produced: " + (value - 1));
                notify();  // Notify consumer
                Thread.sleep(1000);
            }
        }
    }

    public void consume() throws InterruptedException {
        while (true) {
            synchronized (this) {
                while (queue.isEmpty()) {
                    wait();  // Wait for producer to produce
                }
                int val = queue.remove();
                System.out.println("Consumed: " + val);
                notify();  // Notify producer
                Thread.sleep(1000);
            }
        }
    }

    public static void main(String[] args) {
        ProducerConsumer pc = new ProducerConsumer();

        Thread producer = new Thread(() -> {
            try { pc.produce(); } catch (InterruptedException e) { }
        });

        Thread consumer = new Thread(() -> {
            try { pc.consume(); } catch (InterruptedException e) { }
        });

        producer.start();
        consumer.start();
    }
}
```

---

## 3. Producer-Consumer with BlockingQueue (Better)

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class ProducerConsumerBQ {
    private final BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);

    public void produce() throws InterruptedException {
        int value = 0;
        while (true) {
            queue.put(value++);  // Blocks if queue is full
            System.out.println("Produced: " + (value - 1));
            Thread.sleep(1000);
        }
    }

    public void consume() throws InterruptedException {
        while (true) {
            Integer val = queue.take();  // Blocks if queue is empty
            System.out.println("Consumed: " + val);
            Thread.sleep(1000);
        }
    }
}
```

---

## 4. Dining Philosophers Problem

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class DiningPhilosophers {
    private static final int NUM_PHILOSOPHERS = 5;
    private final Lock[] forks = new Lock[NUM_PHILOSOPHERS];

    public DiningPhilosophers() {
        for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
            forks[i] = new ReentrantLock();
        }
    }

    public void dine(int philosopherId) {
        int leftFork = philosopherId;
        int rightFork = (philosopherId + 1) % NUM_PHILOSOPHERS;

        while (true) {
            // Pick up left fork
            forks[leftFork].lock();
            try {
                // Pick up right fork
                if (forks[rightFork].tryLock()) {
                    try {
                        // Eat
                        System.out.println("Philosopher " + philosopherId + " is eating");
                        Thread.sleep(1000);
                    } finally {
                        forks[rightFork].unlock();
                    }
                }
            } finally {
                forks[leftFork].unlock();
            }

            // Think
            System.out.println("Philosopher " + philosopherId + " is thinking");
            Thread.sleep(1000);
        }
    }

    public static void main(String[] args) {
        DiningPhilosophers dp = new DiningPhilosophers();
        for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
            final int id = i;
            new Thread(() -> dp.dine(id)).start();
        }
    }
}
```

---

## 5. Thread-Safe Singleton

```java
// Double-checked locking
public class Singleton {
    private volatile static Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

// Bill Pugh (inner static class) — best approach
public class SingletonBest {
    private SingletonBest() {}

    private static class SingletonHolder {
        private static final SingletonBest INSTANCE = new SingletonBest();
    }

    public static SingletonBest getInstance() {
        return SingletonHolder.INSTANCE;
    }
}

// Enum singleton — most robust
public enum SingletonEnum {
    INSTANCE;

    public void doSomething() {
        // ...
    }
}
```

---

## 6. Read-Write Lock

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReadWriteLockExample {
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private String data = "";

    public String read() {
        lock.readLock().lock();
        try {
            return data;  // Multiple threads can read simultaneously
        } finally {
            lock.readLock().unlock();
        }
    }

    public void write(String newData) {
        lock.writeLock().lock();
        try {
            data = newData;  // Only one thread can write
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

---

## 7. Semaphore — Rate Limiter

```java
import java.util.concurrent.Semaphore;

public class RateLimiter {
    private final Semaphore semaphore;

    public RateLimiter(int maxConcurrentRequests) {
        this.semaphore = new Semaphore(maxConcurrentRequests);
    }

    public void execute(Runnable task) throws InterruptedException {
        semaphore.acquire();  // Acquire a permit
        try {
            task.run();
        } finally {
            semaphore.release();  // Release the permit
        }
    }

    public static void main(String[] args) throws InterruptedException {
        RateLimiter limiter = new RateLimiter(3);  // Max 3 concurrent

        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            new Thread(() -> {
                try {
                    limiter.execute(() -> {
                        System.out.println("Task " + taskId + " running");
                        Thread.sleep(1000);
                    });
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }).start();
        }
    }
}
```

---

## 8. CountDownLatch — Wait for Multiple Threads

```java
import java.util.concurrent.CountDownLatch;

public class CountDownLatchExample {
    public static void main(String[] args) throws InterruptedException {
        int numServices = 3;
        CountDownLatch latch = new CountDownLatch(numServices);

        for (int i = 0; i < numServices; i++) {
            new Thread(() -> {
                try {
                    System.out.println("Service starting...");
                    Thread.sleep(2000);
                    System.out.println("Service ready");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown();  // Signal completion
                }
            }).start();
        }

        // Wait for all services to be ready
        latch.await();
        System.out.println("All services ready — starting main application");
    }
}
```

---

## 9. CyclicBarrier — Synchronized Start

```java
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierExample {
    public static void main(String[] args) {
        int numThreads = 3;
        CyclicBarrier barrier = new CyclicBarrier(numThreads, () -> {
            System.out.println("All threads reached the barrier — starting together!");
        });

        for (int i = 0; i < numThreads; i++) {
            final int threadId = i;
            new Thread(() -> {
                try {
                    System.out.println("Thread " + threadId + " is ready");
                    barrier.await();  // Wait for all threads
                    System.out.println("Thread " + threadId + " is running");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

---

## 10. CompletableFuture Chain

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;

public class CompletableFutureChain {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        CompletableFuture<String> result = CompletableFuture
            .supplyAsync(() -> {
                System.out.println("Step 1: Fetching user");
                return "Alice";
            })
            .thenApplyAsync(user -> {
                System.out.println("Step 2: Enriching user: " + user);
                return user + " (VIP)";
            })
            .thenApplyAsync(user -> {
                System.out.println("Step 3: Formatting: " + user);
                return "Hello, " + user + "!";
            })
            .exceptionally(ex -> {
                System.err.println("Error: " + ex.getMessage());
                return "Hello, Guest!";
            });

        System.out.println("Final result: " + result.get());
    }
}
```

---

## Key Patterns

| Pattern | Tool | Use Case |
|---------|------|----------|
| | Mutual exclusion | `synchronized`, `ReentrantLock` | Protect shared state |
| | Thread coordination | `wait/notify`, `BlockingQueue` | Producer-consumer |
| | Read-write separation | `ReadWriteLock` | Read-heavy workloads |
| | Resource limiting | `Semaphore` | Rate limiting, connection pools |
| | Synchronization point | `CountDownLatch` | Wait for N tasks to complete |
| | Reusable barrier | `CyclicBarrier` | Synchronized start of multiple threads |
| | Async composition | `CompletableFuture` | Chaining async operations |
| | Atomic operations | `AtomicInteger`, etc. | Lock-free counters |

---

## Interview Questions

### 1. What is the difference between a Process and a Thread?

| Aspect | Process | Thread |
|--------|---------|--------|
| **Definition** | Independent execution unit with its own memory space | Lightweight execution unit within a process |
| **Memory** | Own address space, heap, stack | Shares process memory (heap), has own stack |
| **Creation** | Heavy (OS-level, copies memory) | Light (within existing process) |
| **Communication** | IPC (pipes, sockets, shared memory) | Direct (shared memory) |
| **Context Switch** | Slow (OS kernel mode) | Fast (user mode) |
| **Fault Tolerance** | Crash doesn't affect others | Crash can bring down entire process |
| **Overhead** | High (MB of memory per process) | Low (KB of stack per thread) |

**Key Point**: Threads are "lightweight processes" that share the same memory space but have independent execution stacks.

---

### 2. What is the difference between start() and run()?

```java
Thread t = new Thread(() -> System.out.println("Running"));

t.start();  // Creates NEW thread, calls run() in that thread
t.run();    // Executes in CURRENT thread (no new thread created!)
```

| Method | What Happens | Thread Created? |
|--------|--------------|-----------------|
| `start()` | JVM creates new OS thread, then calls `run()` in that thread | ✅ Yes |
| `run()` | Just calls the method directly, like any normal method | ❌ No |

**Common Mistake**: Calling `run()` directly instead of `start()` — the code runs synchronously in the calling thread, defeating the purpose of multithreading.

---

### 3. What happens internally when Thread.start() is called?

```
1. JVM creates a new native OS thread
2. Allocates a new stack (typically 1MB, configurable with -Xss)
3. Thread enters "RUNNABLE" state
4. Thread scheduler picks it up for execution
5. JVM calls the thread's run() method in the new thread
6. When run() completes, thread enters "TERMINATED" state
```

**Key Points**:
- `start()` is a native method that delegates to the OS
- The thread is not immediately running — it's scheduled by the OS
- You can only call `start()` once. Calling it again throws `IllegalThreadStateException`

---

### 4. What is the difference between synchronized method and synchronized block?

```java
// Synchronized METHOD — locks the ENTIRE method
public synchronized void transfer(Account from, Account to, int amount) {
    // Entire method is locked — no other thread can enter ANY synchronized method
    // on this object while this is executing
    from.debit(amount);
    to.credit(amount);
}

// Synchronized BLOCK — locks only a specific object
public void transfer(Account from, Account to, int amount) {
    // Only lock what's necessary
    synchronized (from) {
        from.debit(amount);
    }
    synchronized (to) {
        to.credit(amount);
    }
}
```

| Aspect | Synchronized Method | Synchronized Block |
|--------|---------------------|-------------------|
| **Lock Scope** | Entire method body | Only the block |
| **Lock Object** | `this` (or Class object for static) | Any object you choose |
| **Granularity** | Coarse-grained | Fine-grained |
| **Performance** | Slower (locks more) | Faster (locks less) |
| **Flexibility** | Low | High |

**Best Practice**: Use synchronized blocks for better granularity and performance.

---

### 5. What happens when two threads access a synchronized method on the same object?

```java
public class Counter {
    private int count = 0;

    public synchronized void increment() {
        count++;  // Only ONE thread can execute this at a time
    }
}

// Thread A and Thread B both call counter.increment()
// Result: Only one thread enters at a time — no race condition
```

**What Happens**:
1. Thread A enters `increment()` — acquires the object's monitor (lock)
2. Thread B tries to enter `increment()` — **blocks** (waits for monitor)
3. Thread A completes — releases the monitor
4. Thread B acquires the monitor — enters `increment()`
5. Thread B completes — releases the monitor

**Key Point**: The lock is on the **object instance**, not the method. Two different `Counter` objects can have their `increment()` methods executed simultaneously.

---

### 6. Why is volatile not enough for count++?

```java
public class Counter {
    private volatile int count = 0;

    public void increment() {
        count++;  // NOT atomic! Still has race condition
    }
}
```

**The Problem**: `count++` is **not a single operation**. It's three operations:
1. **Read** — `count` from memory into register
2. **Modify** — increment the register value
3. **Write** — write the register value back to memory

```java
// What count++ actually does:
int temp = count;    // Read
temp = temp + 1;     // Modify
count = temp;        // Write
```

**Race Condition Scenario**:
```
Thread A: reads count = 5
Thread B: reads count = 5
Thread A: increments to 6, writes 6
Thread B: increments to 6, writes 6
Result: count = 6 (should be 7!)
```

**volatile only guarantees**:
- **Visibility**: Changes are immediately visible to other threads
- **Ordering**: Prevents instruction reordering

**volatile does NOT guarantee**:
- **Atomicity**: Read-modify-write is still not atomic

**Solution**: Use `AtomicInteger` or `synchronized`:
```java
// Option 1: AtomicInteger (lock-free, preferred)
private AtomicInteger count = new AtomicInteger(0);
public void increment() {
    count.incrementAndGet();  // Atomic!
}

// Option 2: synchronized
public synchronized void increment() {
    count++;
}
```

---

### 7. What is the difference between atomicity, visibility, and ordering?

These are the three guarantees provided by the Java Memory Model (JMM):

| Concept | Definition | Example |
|---------|------------|---------|
| **Atomicity** | Operations complete entirely or not at all — no intermediate state visible | `count++` is NOT atomic (read + modify + write) |
| **Visibility** | When one thread modifies a variable, other threads see the change | `volatile` ensures visibility |
| **Ordering** | The JVM/CPU may reorder instructions for optimization — but within a single thread, the result appears sequential | `synchronized` and `volatile` prevent reordering |

**Detailed Explanation**:

**Atomicity**:
```java
// Without atomicity — intermediate state visible
// Thread A: writing 64-bit long (on 32-bit JVM)
// Thread B: reads first 32 bits (old value), then second 32 bits (new value)
// Result: corrupted value!
long value = 0x123456789ABCDEF0L;
// Thread A writes: first 32 bits = 0x12345678, second 32 bits = 0x9ABCDEF0
// Thread B reads: 0x12345678 + 0x00000000 = corrupted!
```

**Visibility**:
```java
// Without visibility — thread never sees the change
// Thread A:
flag = true;  // Written to CPU cache, not main memory

// Thread B:
while (!flag) { }  // Never exits! flag is still false in its cache
```

**Ordering**:
```java
// Without ordering — instructions reordered by JIT compiler
int a = 1;
int b = 2;
// JIT might reorder to: b = 2; a = 1; (same result in single thread)
// But in multithreaded context, this can cause issues

// With synchronized/volatile — ordering guaranteed
```

---

### 8. What is the difference between wait(), sleep(), and join()?

| Method | Belongs To | Releases Lock? | Purpose | InterruptedException |
|--------|-----------|----------------|---------|---------------------|
| `wait()` | `Object` | ✅ Yes | Wait for notification from another thread | ✅ Yes |
| `sleep()` | `Thread` | ❌ No | Pause execution for specified time | ✅ Yes |
| `join()` | `Thread` | ❌ No | Wait for another thread to die | ✅ Yes |

**Code Examples**:

```java
// wait() — must be called inside synchronized block
synchronized (lock) {
    lock.wait(1000);  // Wait up to 1 second for notification
    // Lock is RELEASED while waiting
}

// sleep() — static method, always pauses current thread
Thread.sleep(1000);  // Pause for 1 second
// Lock is NOT released

// join() — wait for another thread to finish
Thread t = new Thread(() -> { /* work */ });
t.start();
t.join();  // Wait for t to die
// Lock is NOT released
```

**When to Use**:
- `wait()`: Producer-consumer patterns, waiting for a condition
- `sleep()`: Simple pause, retry with delay
- `join()`: Waiting for a thread to complete (e.g., waiting for worker threads)

---

### 9. Why must wait() and notify() be called while holding the object's monitor?

```java
// CORRECT — inside synchronized block
synchronized (lock) {
    lock.wait();  // ✅ Releases lock and waits
    lock.notify();  // ✅ Notifies waiting thread
}

// WRONG — outside synchronized block
lock.wait();  // ❌ IllegalMonitorStateException!
```

**Reason**: `wait()` and `notify()` are **inter-thread communication** mechanisms. They must be called while holding the monitor to:

1. **Prevent race conditions**: If `notify()` is called before `wait()`, the waiting thread would wait forever. Holding the monitor ensures atomicity of the check-and-wait pattern.

2. **Maintain the happens-before relationship**: The monitor lock establishes a memory barrier, ensuring visibility of changes.

3. **Avoid missed signals**: The classic pattern is:
```java
synchronized (lock) {
    while (!condition) {  // Check condition inside synchronized
        lock.wait();       // Release lock and wait
    }
    // Condition is true — proceed
}
```

**What happens if you call wait() without holding the monitor?**
- `IllegalMonitorStateException` is thrown at runtime

---

### 10. What happens if notify() is called when no thread is waiting?

```java
synchronized (lock) {
    lock.notify();  // No thread is waiting — nothing happens
}
// No exception, no error — the notification is simply lost
```

**What Happens**:
- The notification is **lost** — no thread receives it
- No exception is thrown
- The program continues normally

**Why This Matters**:
```java
// BAD — notification can be lost
synchronized (lock) {
    // Thread A checks condition, it's false
    // Thread B changes condition and calls notify()
    // Thread A calls wait() — but notification was already sent!
    // Thread A waits forever
}

// GOOD — use while loop to recheck condition
synchronized (lock) {
    while (!condition) {  // Recheck after waking up
        lock.wait();
    }
}
```

**Key Point**: Always use `wait()` inside a `while` loop to handle spurious wakeups and lost notifications.

---

### 11. What is a spurious wakeup, and why should wait() generally be used inside a while loop?

**Spurious Wakeup**: A thread can wake up from `wait()` **without being notified**, due to:
- OS-level thread scheduling
- JVM implementation details
- Hardware interrupts

```java
// WRONG — if spurious wakeup occurs, condition might still be false
synchronized (lock) {
    if (!condition) {
        lock.wait();  // Wakes up — but condition might still be false!
    }
    // Proceed with assumption that condition is true — WRONG!
}

// CORRECT — recheck condition after waking up
synchronized (lock) {
    while (!condition) {  // Recheck in loop
        lock.wait();  // If spurious wakeup, loop continues
    }
    // Now we're sure condition is true
}
```

**Why while loop?**
1. **Spurious wakeups**: Thread can wake up without `notify()`
2. **Lost notifications**: `notify()` might have been called before `wait()`
3. **Multiple waiting threads**: Another thread might have consumed the notification

**The Pattern**:
```java
synchronized (lock) {
    while (condition not met) {
        lock.wait();  // Release lock, wait, reacquire lock
    }
    // Condition is guaranteed to be true here
}
```

---

### 12. What is the difference between synchronized and ReentrantLock?

| Feature | synchronized | ReentrantLock |
|---------|--------------|---------------|
| **Implementation** | Built-in Java keyword | Class in `java.util.concurrent.locks` |
| **Reentrancy** | Yes (implicit) | Yes (explicit) |
| **Fairness** | No (unfair by default) | Yes (configurable) |
| **Try Lock** | No | Yes (`tryLock()`) |
| **Interruptible Lock** | No | Yes (`lockInterruptibly()`) |
| **Multiple Conditions** | No (single monitor) | Yes (`newCondition()`) |
| **Performance** | Lower overhead (biased locking) | Slightly higher overhead |
| **Release** | Automatic (JVM releases on exit) | Manual (must call `unlock()`) |

**Code Comparison**:

```java
// synchronized — simple, automatic
public synchronized void transfer(Account from, Account to, int amount) {
    from.debit(amount);
    to.credit(amount);
}

// ReentrantLock — more control
private final ReentrantLock lock = new ReentrantLock();

public void transfer(Account from, Account to, int amount) {
    lock.lock();
    try {
        from.debit(amount);
        to.credit(amount);
    } finally {
        lock.unlock();  // MUST release in finally!
    }
}
```

**When to Use ReentrantLock**:
- Need tryLock() with timeout
- Need interruptible lock acquisition
- Need multiple condition variables
- Need fair locking (FIFO order)

**When to Use synchronized**:
- Simple locking needs
- Prefer simplicity over advanced features
- Performance is critical (biased locking optimization)

---

### 13. What is a deadlock? How would you identify one in production?

**Deadlock**: Two or more threads are blocked forever, each waiting for the other to release a lock.

```java
// Thread 1 holds lock A, waits for lock B
// Thread 2 holds lock B, waits for lock A
// Both wait forever — DEADLOCK!

Thread 1:
synchronized (lockA) {
    synchronized (lockB) {  // Waits for Thread 2 to release lockB
        // ...
    }
}

Thread 2:
synchronized (lockB) {
    synchronized (lockA) {  // Waits for Thread 1 to release lockA
        // ...
    }
}
```

**Deadlock Conditions (Coffman Conditions)**:
1. **Mutual Exclusion**: Only one thread can hold a resource at a time
2. **Hold and Wait**: Thread holds one resource while waiting for another
3. **No Preemption**: Resources cannot be forcibly taken from threads
4. **Circular Wait**: Threads form a circular chain waiting for each other

**How to Identify in Production**:

```bash
# 1. Thread dump analysis
jstack <pid> > thread_dump.txt
# Look for "Found one Java-level deadlock" in the output

# 2. VisualVM / JConsole
# Threads tab — look for threads in BLOCKED state

# 3. Programmatic detection
ThreadMXBean bean = ManagementFactory.getThreadMXBean();
long[] deadlockedThreads = bean.findDeadlockedThreads();
if (deadlockedThreads != null) {
    System.err.println("Deadlock detected!");
}

# 4. Log analysis
# Look for threads stuck in BLOCKED state for extended periods
```

**Prevention**:
- **Lock ordering**: Always acquire locks in the same order
- **Lock timeout**: Use `tryLock(timeout)` instead of blocking
- **Avoid nested locks**: Minimize holding multiple locks simultaneously

---

### 14. What is the difference between deadlock, livelock, and starvation?

| Concept | Definition | Example | Outcome |
|---------|------------|---------|---------|
| **Deadlock** | Threads blocked forever, waiting for each other | Thread A holds lock1, waits for lock2; Thread B holds lock2, waits for lock1 | No progress — threads stuck |
| **Livelock** | Threads are active but keep changing state in response to each other, making no progress | Two threads keep yielding to each other: "you go first" "no, you go first" | No progress — but threads are running |
| **Starvation** | A thread never gets access to a shared resource because other threads keep taking it | Low-priority thread never gets CPU time | No progress for the starved thread |

**Code Examples**:

```java
// LIVELOCK — threads keep yielding to each other
public class LivelockExample {
    private final Object lock = new Object();
    private boolean flag = false;

    public void thread1() {
        while (true) {
            synchronized (lock) {
                if (flag) {
                    lock.wait();  // Yield to thread2
                } else {
                    flag = true;
                    lock.notify();
                }
            }
        }
    }

    public void thread2() {
        while (true) {
            synchronized (lock) {
                if (!flag) {
                    lock.wait();  // Yield to thread1
                } else {
                    flag = false;
                    lock.notify();
                }
            }
        }
    }
}

// STARVATION — low-priority thread never gets CPU
Thread lowPriority = new Thread(() -> {
    while (true) {
        // This thread rarely gets CPU time
        // because high-priority threads keep running
    }
});
lowPriority.setPriority(Thread.MIN_PRIORITY);  // Priority 1
lowPriority.start();

Thread highPriority = new Thread(() -> {
    while (true) {
        // This thread hogs the CPU
    }
});
highPriority.setPriority(Thread.MAX_PRIORITY);  // Priority 10
highPriority.start();
```

---

### 15. What is ExecutorService, and why is it preferred over manually creating threads?

**ExecutorService** is a high-level abstraction for managing thread pools and asynchronous task execution.

**Problems with Manual Thread Creation**:
```java
// BAD — manual thread creation
for (int i = 0; i < 1000; i++) {
    new Thread(() -> processRequest()).start();
}
// Problems:
// 1. Each thread consumes ~1MB stack memory → 1000 threads = ~1GB
// 2. No reuse — threads created and destroyed (expensive)
// 3. No throttling — all 1000 threads run simultaneously
// 4. No queuing — no way to queue tasks when all threads busy
// 5. No error handling — uncaught exceptions kill threads silently
// 6. No lifecycle management — no graceful shutdown
```

**ExecutorService Solution**:
```java
// GOOD — thread pool
ExecutorService executor = Executors.newFixedThreadPool(10);
for (int i = 0; i < 1000; i++) {
    executor.submit(() -> processRequest());
}
executor.shutdown();
// Benefits:
// 1. Thread reuse — 10 threads handle 1000 tasks
// 2. Resource control — fixed pool size
// 3. Task queuing — excess tasks wait in queue
// 4. Error handling — exceptions captured in Future
// 5. Lifecycle management — graceful shutdown
```

**Key Benefits**:
- **Thread reuse**: Avoids creation/destruction overhead
- **Resource management**: Limits concurrent threads
- **Task queuing**: Handles overload gracefully
- **Error handling**: Centralized exception handling
- **Monitoring**: Built-in metrics (active count, queue size)

---

### 16. What is the difference between execute() and submit()?

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

// execute() — fire and forget, no return value
executor.execute(() -> {
    System.out.println("Task executed");
    // If exception occurs, thread dies silently
});

// submit() — returns Future for result/exception handling
Future<?> future = executor.submit(() -> {
    System.out.println("Task submitted");
    return "result";
});

// Get result (blocks until complete)
try {
    Object result = future.get();
} catch (ExecutionException e) {
    // Exception is wrapped and can be retrieved
    Throwable cause = e.getCause();
}
```

| Feature | execute() | submit() |
|---------|-----------|----------|
| **Return Type** | `void` | `Future<?>` or `Future<T>` |
| **Exception Handling** | Uncaught exceptions kill thread silently | Exceptions captured in Future |
| **Result Retrieval** | No | Yes, via `future.get()` |
| **Use Case** | Fire-and-forget tasks | Tasks with return values or error handling |

**When to Use**:
- `execute()`: Simple tasks where you don't need the result
- `submit()`: Tasks where you need the result or want to handle exceptions

---

### 17. What happens when a task submitted to ExecutorService throws an exception?

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

// With execute() — exception kills the thread silently
executor.execute(() -> {
    throw new RuntimeException("Task failed!");
    // Thread dies, new thread is created to replace it
    // No way to know the task failed!
});

// With submit() — exception is captured in Future
Future<?> future = executor.submit(() -> {
    throw new RuntimeException("Task failed!");
});

try {
    future.get();  // Throws ExecutionException
} catch (ExecutionException e) {
    Throwable cause = e.getCause();  // Get the original exception
    System.err.println("Task failed: " + cause.getMessage());
}
```

**Key Points**:
- `execute()`: Exception kills the thread, but the pool creates a replacement
- `submit()`: Exception is wrapped in `ExecutionException` and can be retrieved
- The thread pool **does not die** — it continues processing other tasks

---

### 18. How can an incorrectly configured thread pool cause high API latency even when CPU usage is low?

**Scenario**: CPU is 30%, but response time increased from 200ms to 5 seconds.

**Root Causes**:

```java
// Problem 1: Pool too small — tasks queue up
ExecutorService executor = Executors.newFixedThreadPool(2);
// 100 requests arrive, only 2 threads process them
// 98 requests wait in queue → high latency!

// Problem 2: Unbounded queue — tasks pile up
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4, 10, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>()  // Unbounded queue!
);
// Queue grows indefinitely → memory pressure → GC pauses → latency

// Problem 3: Wrong rejection policy
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4, 4, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(10),
    new ThreadPoolExecutor.AbortPolicy()  // Throws exception!
);
// When queue is full, tasks are rejected → 503 errors

// Problem 4: Blocking operations in tasks
executor.submit(() -> {
    Thread.sleep(5000);  // Thread blocked for 5 seconds
    // Other tasks wait for this thread to become available
});
```

**Diagnosis**:
```java
ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newFixedThreadPool(4);

// Monitor these metrics:
System.out.println("Active threads: " + executor.getActiveCount());
System.out.println("Queue size: " + executor.getQueue().size());
System.out.println("Pool size: " + executor.getPoolSize());
```

**Solutions**:
- Size pool correctly: `CPU cores × 2 + disk spindles`
- Use bounded queue with appropriate rejection policy
- Avoid blocking operations in thread pool tasks
- Use `CallerRunsPolicy` to apply backpressure

---

### 19. What is the difference between AtomicInteger, synchronized, and LongAdder?

| Feature | AtomicInteger | synchronized | LongAdder |
|---------|---------------|--------------|-----------|
| **Mechanism** | CAS (Compare-And-Swap) | Monitor lock | Striped cells (reduced contention) |
| **Performance** | Good for low contention | Poor under high contention | Excellent under high contention |
| **Blocking** | Non-blocking | Blocking | Non-blocking |
| **Memory** | Single variable | Single variable | Multiple cells (more memory) |
| **Use Case** | General purpose counters | Complex critical sections | High-contention counters |

**Code Examples**:

```java
// AtomicInteger — CAS-based, good for most cases
private AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();  // Lock-free, uses CPU CAS instruction

// synchronized — blocking, good for complex operations
private int counter = 0;
public synchronized void increment() {
    counter++;
}

// LongAdder — striped cells, best for high contention
private LongAdder counter = new LongAdder();
counter.increment();  // Spreads updates across cells
long total = counter.sum();  // Sum all cells
```

**When to Use Each**:
- **AtomicInteger**: Low to medium contention, simple counters
- **synchronized**: Complex operations requiring multiple variables
- **LongAdder**: High contention (many threads updating same counter)

**Performance Comparison** (under high contention):
```
AtomicInteger: ~1000 ops/ms (CAS retries cause overhead)
synchronized: ~100 ops/ms (threads block and wait)
LongAdder: ~5000 ops/ms (striped cells reduce contention)
```

---

### 20. Production scenario: Your API receives 1,000 requests simultaneously. CPU is only 30%, but response time increases from 200 ms to 5 seconds. How would you determine whether the bottleneck is the thread pool, database connection pool, lock contention, or blocking I/O?

**Diagnostic Approach**:

```java
// Step 1: Check thread pool metrics
ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newFixedThreadPool(10);
System.out.println("Active threads: " + executor.getActiveCount());
System.out.println("Queue size: " + executor.getQueue().size());
System.out.println("Pool size: " + executor.getPoolSize());

// If queue size is large → thread pool is the bottleneck
// If active threads == pool size → all threads busy

// Step 2: Check database connection pool
HikariDataSource ds = (HikariDataSource) dataSource;
HikariPoolMXBean pool = ds.getHikariPoolMXBean();
System.out.println("Active connections: " + pool.getActiveConnections());
System.out.println("Threads awaiting: " + pool.getThreadsAwaitingConnection());

// If threads awaiting > 0 → connection pool is the bottleneck

// Step 3: Check for lock contention
ThreadMXBean bean = ManagementFactory.getThreadMXBean();
long[] deadlocked = bean.findDeadlockedThreads();
if (deadlocked != null) {
    System.err.println("Deadlock detected!");
}

// Check thread states — many BLOCKED threads = lock contention
ThreadInfo[] threads = bean.dumpAllThreads(true, true);
for (ThreadInfo thread : threads) {
    if (thread.getThreadState() == Thread.State.BLOCKED) {
        System.err.println("Blocked: " + thread.getThreadName() +
            " waiting for " + thread.getLockInfo());
    }
}

// Step 4: Check for blocking I/O
// Use profiler (VisualVM, async-profiler) to see thread states
// If many threads are in WAITING or TIMED_WAITING → blocking I/O

// Step 5: Distributed tracing
// Use OpenTelemetry/Jaeger to trace request flow
// Identify which component is slow
```

**Decision Matrix**:

| Symptom | Likely Bottleneck | Action |
|---------|-------------------|--------|
| Thread pool queue growing | Thread pool too small | Increase pool size or optimize tasks |
| DB connection pool exhausted | Connection pool too small | Increase pool size, fix leaks |
| Many threads BLOCKED | Lock contention | Reduce lock scope, use finer-grained locks |
| Threads in WAITING state | Blocking I/O | Use async I/O, increase timeouts |
| CPU low, latency high | Thread pool or I/O bottleneck | Profile threads, check queue sizes |

**Quick Checks**:
```bash
# 1. Thread dump — look for BLOCKED threads
jstack <pid> | grep -c "BLOCKED"

# 2. Check thread pool queue
# If using Spring Boot:
curl http://localhost:8080/actuator/metrics/executor.pool.size

# 3. Check DB connection pool
curl http://localhost:8080/actuator/metrics/hikaricp.connections.active

# 4. Check for lock contention
# Use Java Flight Recorder or async-profiler
```

**Key Takeaway**: Low CPU with high latency almost always indicates **thread contention** — either threads are waiting for locks, waiting for I/O, or queued in a thread pool. Use thread dumps, metrics, and distributed tracing to identify the exact bottleneck.
