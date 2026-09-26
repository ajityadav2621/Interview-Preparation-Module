# Java Locks: synchronized vs ReentrantLock

## Core Differences

| Feature | synchronized | ReentrantLock |
|---------|-------------|---------------|
| **API** | Built-in keyword | Explicit API |
| **Fairness** | No | Configurable |
| **Condition** | Single wait set | Multiple conditions |
| **Try Lock** | No | Yes (tryLock) |
| **Unlock** | Automatic | Manual (must call unlock) |
| **Performance** | Similar (Java 6+) | Similar |

## When to Use Each

### Use synchronized When:
- **Simple synchronization** - basic use cases
- **Automatic unlock** - no risk of forgetting
- **Code simplicity** - less boilerplate
- **Final methods** - cannot override

```java
// Simple method synchronization
public synchronized void setValue(String value) {
    this.value = value;
}
```

### Use ReentrantLock When:
- **Fairness required** - FIFO ordering
- **Try lock needed** - non-blocking acquisition
- **Multiple conditions** - different wait sets
- **Interruptible lock acquisition**
- **Performance tuning** - configurable

```java
private final ReentrantLock lock = new ReentrantLock(true); // Fair lock

public void setValue(String value) {
    lock.lock();
    try {
        this.value = value;
    } finally {
        lock.unlock(); // Must be in finally
    }
}

// Try lock with timeout
if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
    try {
        // Critical section
    } finally {
        lock.unlock();
    }
}
```

## ReentrantLock Advantages

### 1. **Fairness**
```java
ReentrantLock fairLock = new ReentrantLock(true);
// Threads acquire lock in FIFO order
```

### 2. **Multiple Conditions**
```java
ReentrantLock lock = new ReentrantLock();
Condition notFull = lock.newCondition();
Condition notEmpty = lock.newCondition();

// Similar to BlockingQueue implementation
```

### 3. **Interruptible Locking**
```java
lock.lockInterruptibly(); // Responds to interrupts
```

## Interview Tip
Explain that for most use cases, synchronized is sufficient. Use ReentrantLock when you need advanced features like fairness, tryLock, or multiple conditions.