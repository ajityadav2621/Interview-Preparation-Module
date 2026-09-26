# Identifying Deadlock in Java Applications

## What is Deadlock?
A situation where two or more threads are blocked forever, each waiting for the other to release a resource.

## Classic Example
```java
// Thread 1
synchronized(lockA) {
    synchronized(lockB) { // Waits for lockB held by Thread 2
        // ...
    }
}

// Thread 2
synchronized(lockB) {
    synchronized(lockA) { // Waits for lockA held by Thread 1
        // ...
    }
}
```

## How to Identify Deadlock

### 1. **JVM Tools**
```bash
# Get thread dump
jstack <pid> > thread_dump.txt

# Look for "Deadlock" or "BLOCKED" state
jstack -l <pid> | grep -A 10 "Deadlock"
```

### 2. **JConsole / JVisualVM**
- Connect to running application
- Threads tab shows deadlocked threads
- Visual representation of lock relationships

### 3. **Programmatic Detection**
```java
ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();
long[] deadlockedThreads = mxBean.findDeadlockedThreads();

if (deadlockedThreads != null) {
    System.out.println("Deadlock detected!");
    ThreadInfo[] threadInfos = mxBean.getThreadInfo(deadlockedThreads);
    for (ThreadInfo info : threadInfos) {
        System.out.println(info);
    }
}
```

### 4. **Production Detection**
- Monitor thread states (BLOCKED, WAITING)
- Set up alerts for high thread blocking
- Use APM tools (Dynatrace, AppDynamics)
- Check for increasing CPU with no progress

## Prevention Strategies
- Lock ordering (always acquire locks in same order)
- Timeout on lock acquisition (tryLock with timeout)
- Use higher-level concurrency utilities
- Avoid nested synchronization when possible

## Interview Tip
Mention that deadlocks are hard to reproduce in testing but easy to detect in production with thread dumps. Practice analyzing thread dumps to identify deadlocked threads.