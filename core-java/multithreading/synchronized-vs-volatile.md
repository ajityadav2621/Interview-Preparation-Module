# synchronized vs volatile — When Would You Use Each?

## Overview

Both `synchronized` and `volatile` are Java keywords used in multithreading, but they serve **different purposes** and have **different capabilities**.

## Key Differences

| Feature | `synchronized` | `volatile` |
|---------|---------------|------------|
| **Scope** | Block/method level | Variable level |
| **Mutual Exclusion** | Yes — only one thread can enter | No — multiple threads can access |
| **Visibility** | Yes (via memory barrier) | Yes (via memory barrier) |
| **Atomicity** | Yes (compound operations) | No (only single reads/writes) |
| **Blocking** | Yes — thread blocks if lock is held | No — thread never blocks |
| **Performance** | Slower (OS-level locking) | Faster (memory barrier only) |
| **Reentrancy** | Yes | N/A |
| **Null handling** | N/A | Can be applied to any variable |

## What `volatile` Solves

### The Visibility Problem

Without `volatile`, a thread may cache a variable's value in its local CPU cache and not see updates from other threads:

```java
// WITHOUT volatile — BROKEN
public class VisibilityExample {
    private boolean running = true;  // Not volatile

    public void doWork() {
        while (running) {
            // Do some work
        }
        System.out.println("Stopped");
    }

    public void stop() {
        running = false;  // Other thread may never see this!
    }
}
```

### With `volatile` — Fixed

```java
public class VisibilityExample {
    private volatile boolean running = true;  // Volatile ensures visibility

    public void doWork() {
        while (running) {
            // Do some work
        }
        System.out.println("Stopped");
    }

    public void stop() {
        running = false;  // Other thread will see this update
    }
}
```

### How `volatile` Works

When a variable is declared `volatile`:

1. **Read**: The thread reads directly from **main memory**, not from its local cache
2. **Write**: The thread writes directly to **main memory**, not to its local cache
3. **Memory barrier**: A memory barrier is inserted, preventing instruction reordering

```java
// What happens with volatile:
// Thread 1: write to volatile variable → flush to main memory
// Thread 2: read from volatile variable → load from main memory
```

## What `synchronized` Solves

### The Atomicity Problem

`volatile` only ensures visibility of **single** reads/writes. It does **not** make compound operations atomic:

```java
// WITHOUT synchronized — BROKEN
public class Counter {
    private volatile int count = 0;

    public void increment() {
        count++;  // NOT atomic! This is: read → increment → write
    }
}
```

The `count++` operation involves three steps:
1. Read `count` from memory
2. Increment the value
3. Write the new value back to memory

Even with `volatile`, two threads can interleave these steps:

```
Thread 1: read count (0)
Thread 2: read count (0)
Thread 1: increment → 1
Thread 2: increment → 1
Thread 1: write 1
Thread 2: write 1
// Result: count = 1, but should be 2!
```

### With `synchronized` — Fixed

```java
public class Counter {
    private int count = 0;

    public synchronized void increment() {
        count++;  // Atomic — only one thread can execute this at a time
    }

    // Or using a synchronized block:
    public void incrementWithBlock() {
        synchronized (this) {
            count++;
        }
    }
}
```

## When to Use Each

### Use `volatile` when:

1. **Simple flags** — a boolean flag that signals state changes
2. **Visibility is the only concern** — no compound operations
3. **Read-heavy, write-rare** scenarios
4. **Performance is critical** — volatile has lower overhead

```java
// Good use cases for volatile:
public class Worker {
    private volatile boolean shutdownRequested = false;
    private volatile long lastProcessedId;

    public void shutdown() {
        shutdownRequested = true;
    }

    public void doWork() {
        while (!shutdownRequested) {
            // Process items
            lastProcessedId = getCurrentId();
        }
    }
}
```

### Use `synchronized` when:

1. **Compound operations** — read-modify-write sequences
2. **Mutual exclusion** — only one thread should access a resource
3. **Critical sections** — protecting shared mutable state
4. **Atomic check-then-act** operations

```java
// Good use cases for synchronized:
public class BankAccount {
    private int balance;

    public synchronized void deposit(int amount) {
        balance += amount;  // Compound operation
    }

    public synchronized void withdraw(int amount) {
        if (balance >= amount) {  // Check-then-act
            balance -= amount;
        }
    }

    public synchronized int getBalance() {
        return balance;
    }
}
```

## Can They Be Used Together?

Yes! `volatile` for visibility, `synchronized` for atomicity:

```java
public class Counter {
    private volatile int count = 0;  // Visibility
    private final Object lock = new Object();

    public void increment() {
        synchronized (lock) {  // Atomicity
            count++;
        }
    }

    public int getCount() {
        return count;  // Volatile read — always sees latest value
    }
}
```

## Double-Checked Locking Pattern

A classic pattern that uses both:

```java
public class Singleton {
    private volatile static Singleton instance;  // Volatile is critical here!

    public static Singleton getInstance() {
        if (instance == null) {  // First check (no locking)
            synchronized (Singleton.class) {
                if (instance == null) {  // Second check (with locking)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**Why `volatile` is needed**: Without it, the JVM might reorder the object creation and assignment, causing another thread to see a partially constructed object.

## Limitations

### `volatile` Limitations:
- Cannot make compound operations atomic
- Cannot provide mutual exclusion
- Cannot handle check-then-act scenarios

### `synchronized` Limitations:
- Thread blocks if lock is held (performance overhead)
- Can cause deadlocks if not used carefully
- Cannot be applied to individual variables (only blocks/methods)

## Performance Comparison

```java
// volatile: ~1-2ns overhead (memory barrier)
private volatile int flag;

// synchronized: ~10-100ns overhead (OS-level locking)
public synchronized void method() { ... }
```

## Key Takeaway

> Use `volatile` for **simple visibility** of single variable reads/writes. Use `synchronized` for **atomicity** and **mutual exclusion** of compound operations. They solve different problems and are often complementary.
