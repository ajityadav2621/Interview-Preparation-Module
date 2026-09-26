# How Would You Identify a Deadlock in a Java Application?

## What is a Deadlock?

A **deadlock** occurs when two or more threads are blocked forever, each waiting for a lock that another thread in the same deadlock holds.

### Classic Deadlock Example

```java
public class DeadlockExample {
    private static final Object lock1 = new Object();
    private static final Object lock2 = new Object();

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (lock1) {
                System.out.println("Thread 1: Holding lock 1...");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                System.out.println("Thread 1: Waiting for lock 2...");
                synchronized (lock2) {
                    System.out.println("Thread 1: Got lock 2");
                }
            }
        });

        Thread t2 = new Thread(() -> {
            synchronized (lock2) {
                System.out.println("Thread 2: Holding lock 2...");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                System.out.println("Thread 2: Waiting for lock 1...");
                synchronized (lock1) {
                    System.out.println("Thread 2: Got lock 1");
                }
            }
        });

        t1.start();
        t2.start();
        // Both threads will deadlock — neither can proceed
    }
}
```

## How to Identify a Deadlock

### 1. Thread Dump Analysis (Most Common)

The most reliable way to detect a deadlock is to take a **thread dump** and look for the "Found one Java-level deadlock" section.

#### Taking a Thread Dump

**Option A: Using `jstack`**

```bash
# Find the Java process ID
jps

# Take a thread dump
jstack <pid>
```

**Option B: Using `kill -3` (Unix/Linux)**

```bash
kill -3 <pid>
# The thread dump is printed to the application's stdout
```

**Option C: Using JMX**

```java
import java.lang.management.ManagementFactory;
import java.lang.management.ThreadMXBean;

ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
long[] threadIds = threadBean.findDeadlockedThreads();
if (threadIds != null) {
    System.out.println("Deadlock detected!");
    for (long threadId : threadIds) {
        System.out.println("Thread ID: " + threadId);
    }
}
```

#### Reading a Thread Dump

A thread dump showing a deadlock looks like this:

```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f8b4c005b00 (object 0x000000076b5c3a40, a java.lang.Object),
  which is held by "Thread-0"
"Thread-0":
  waiting to lock monitor 0x00007f8b4c006c00 (object 0x000000076b5c3a50, a java.lang.Object),
  which is held by "Thread-1"

   at DeadlockExample.lambda$main$1(DeadlockExample.java:35)
   - waiting to lock <0x000000076b5c3a50> (a java.lang.Object)
   - locked   <0x000000076b5c3a40> (a java.lang.Object)

   at DeadlockExample.lambda$main$0(DeadlockExample.java:22)
   - waiting to lock <0x000000076b5c3a40> (a java.lang.Object)
   - locked   <0x000000076b5c3a50> (a java.lang.Object)
```

### 2. JVM Tools

#### `jconsole`

```bash
jconsole <pid>
# Navigate to the "Threads" tab
# Look for threads in "BLOCKED" state
```

#### `jvisualvm`

```bash
jvisualvm
# Connect to the process
# View thread states and detect deadlocks
```

### 3. Programmatic Detection

```java
import java.lang.management.*;
import java.util.*;

public class DeadlockDetector {
    public static void checkForDeadlocks() {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        
        // Find deadlocked threads
        long[] deadlockedThreadIds = threadBean.findDeadlockedThreads();
        
        if (deadlockedThreadIds != null) {
            ThreadInfo[] threadInfos = threadBean.getThreadInfo(
                deadlockedThreadIds, true, true);
            
            System.out.println("=== DEADLOCK DETECTED ===");
            for (ThreadInfo threadInfo : threadInfos) {
                System.out.println("Thread: " + threadInfo.getThreadName());
                System.out.println("  State: " + threadInfo.getThreadState());
                System.out.println("  Lock: " + threadInfo.getLockName());
                System.out.println("  Lock Owner: " + threadInfo.getLockOwnerName());
                System.out.println();
            }
        } else {
            System.out.println("No deadlocks detected.");
        }
    }
}
```

### 4. Application-Level Monitoring

```java
// Add a watchdog thread that periodically checks for deadlocks
public class DeadlockWatchdog {
    private final Thread watchdog;
    private final ThreadMXBean threadBean;

    public DeadlockWatchdog() {
        threadBean = ManagementFactory.getThreadMXBean();
        watchdog = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    Thread.sleep(5000);  // Check every 5 seconds
                    long[] deadlocked = threadBean.findDeadlockedThreads();
                    if (deadlocked != null) {
                        System.err.println("DEADLOCK DETECTED!");
                        // Log, alert, or take corrective action
                        threadBean.getThreadInfo(deadlocked);
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        });
        watchdog.setDaemon(true);
        watchdog.start();
    }
}
```

## How to Prevent Deadlocks

### 1. Lock Ordering

Always acquire locks in a **consistent order**:

```java
public class LockOrdering {
    private static final Object lock1 = new Object();
    private static final Object lock2 = new Object();

    // Always acquire lock1 before lock2
    public void method1() {
        synchronized (lock1) {
            synchronized (lock2) {
                // Do work
            }
        }
    }

    public void method2() {
        synchronized (lock1) {  // Same order!
            synchronized (lock2) {
                // Do work
            }
        }
    }
}
```

### 2. Timeout-Based Locking

Use `ReentrantLock.tryLock()` with a timeout:

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.TimeUnit;

public class TimeoutLock {
    private final ReentrantLock lock1 = new ReentrantLock();
    private final ReentrantLock lock2 = new ReentrantLock();

    public void doWork() {
        try {
            boolean acquired1 = lock1.tryLock(1, TimeUnit.SECONDS);
            boolean acquired2 = lock2.tryLock(1, TimeUnit.SECONDS);
            
            if (acquired1 && acquired2) {
                // Do work
            } else {
                // Handle timeout — release any acquired locks
                if (acquired1) lock1.unlock();
                if (acquired2) lock2.unlock();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            if (lock1.isHeldByCurrentThread()) lock1.unlock();
            if (lock2.isHeldByCurrentThread()) lock2.unlock();
        }
    }
}
```

### 3. Avoid Nested Locks

Minimize the scope of synchronized blocks and avoid nesting locks when possible.

## Common Deadlock Patterns

### Pattern 1: Lock Ordering Violation

```java
// Thread 1: lock A → lock B
// Thread 2: lock B → lock A
// DEADLOCK!
```

### Pattern 2: Self-Deadlock

```java
public class SelfDeadlock {
    private final Object lock = new Object();

    public synchronized void method1() {
        method2();  // Tries to acquire the same lock — deadlocks!
    }

    public synchronized void method2() {
        // ...
    }
}
```

### Pattern 3: Resource Starvation

```java
// Thread 1 holds lock A and waits for lock B
// Thread 2 holds lock B and waits for lock A
// Neither will ever release their lock
```

## Key Takeaways

1. **Thread dumps** are the primary tool for deadlock detection
2. **`jstack`** and **JMX** can detect deadlocks programmatically
3. **Lock ordering** is the most effective prevention strategy
4. **Timeout-based locks** (`tryLock`) can prevent deadlocks from persisting
5. **Avoid nested locks** when possible
6. **Monitor in production** — add deadlock detection to your monitoring stack

## Related

- See [`synchronized-vs-volatile.md`](synchronized-vs-volatile.md) for lock-related concepts
- See [`exception-in-thread.md`](exception-in-thread.md) for thread failure handling
