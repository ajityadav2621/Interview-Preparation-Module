# CPU Is Only 20%, but Requests Are Timing Out. What Would You Investigate?

## The Problem

Low CPU usage with high latency is a classic symptom of a **resource bottleneck that is NOT CPU**. The application is waiting for something else.

## Investigation Steps

### 1. Check Thread Dumps

```bash
# Take multiple thread dumps 10 seconds apart
jstack <pid> > threaddump1.txt
sleep 10
jstack <pid> > threaddump2.txt
```

**Look for**:
- **BLOCKED threads**: Waiting for locks
- **WAITING threads**: Waiting for I/O, network, or conditions
- **TIMED_WAITING threads**: Sleeping or waiting with timeout

```
"http-nio-8080-BufferedOperator-3" #25 daemon prio=5 os_prio=0 tid=0x00007f8b4c00 
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x000000076b5c3a40> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2046)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1073)
	- locked <0x000000076b5c3a40> (a java.util.concurrent.ThreadPoolExecutor)
```

### 2. Check I/O Wait

```bash
# Check I/O wait time
iostat -x 1

# Check disk I/O
iotop

# Check network I/O
iftop
ss -tuln  # Check open connections
netstat -an | grep ESTABLISHED | wc -l  # Count connections
```

### 3. Check Database Connection Pool

```java
// Monitor connection pool
HikariDataSource ds = (HikariDataSource) dataSource;
HikariPoolMXBean poolBean = ds.getHikariPoolMXBean();

System.out.println("Active connections: " + poolBean.getActiveConnections());
System.out.println("Idle connections: " + poolBean.getIdleConnections());
System.out.println("Total connections: " + poolBean.getTotalConnections());
System.out.println("Threads awaiting connection: " + poolBean.getThreadsAwaitingConnection());
```

**Red flag**: `Threads awaiting connection > 0` means the pool is exhausted.

### 4. Check External Service Latency

```java
// Add timing to external calls
@RestController
public class OrderController {
    
    @GetMapping("/orders/{id}")
    public OrderResponse getOrder(@PathVariable String id) {
        long start = System.currentTimeMillis();
        User user = userService.getUser(order.getUserId());  // May be slow
        long elapsed = System.currentTimeMillis() - start;
        log.info("User service call took {}ms", elapsed);
        
        return new OrderResponse(order, user);
    }
}
```

### 5. Check for Lock Contention

```java
// Thread dump showing lock contention
"http-nio-8080-exec-5" #31 daemon prio=5 os_prio=0 tid=0x00007f8b4c00
   java.lang.Thread.State: BLOCKED
	at com.example.service.OrderService.processOrder(OrderService.java:45)
	- waiting to lock <0x000000076b5c3a40> (a java.lang.Object)
	at com.example.controller.OrderController.createOrder(OrderController.java:30)
```

### 6. Check for Resource Exhaustion

#### File Descriptors
```bash
# Check file descriptor usage
lsof -p <pid> | wc -l
ulimit -n  # Check limit

# If near the limit, increase it
ulimit -n 65536
```

#### Database Connections
```sql
-- Check active connections
SHOW PROCESSLIST;  -- MySQL
SELECT * FROM pg_stat_activity;  -- PostgreSQL
```

#### Thread Pool Exhaustion
```java
// Check if request processing thread pool is full
ThreadPoolExecutor executor = (ThreadPoolExecutor) requestProcessor;
System.out.println("Active threads: " + executor.getActiveCount());
System.out.println("Queue size: " + executor.getQueue().size());
System.out.println("Completed tasks: " + executor.getCompletedTaskCount());
```

### 7. Check for Deadlocks

```bash
# jstack will show deadlocks
jstack <pid> | grep -A 10 "Found one Java-level deadlock"
```

### 8. Check Network Issues

```bash
# Check for network latency
ping -c 10 external-service.com

# Check for packet loss
ping -c 100 external-service.com

# Check DNS resolution time
dig external-service.com
```

## Common Root Causes

### 1. Database Connection Pool Exhaustion
```yaml
# Pool too small for the load
spring:
  datasource:
    hikari:
      maximum-pool-size: 10  # Too small!
```

### 2. External Service Latency
```java
// No timeout configured — waits forever
User user = userClient.getUser(userId);  // External call with no timeout

// Fix: Add timeout
User user = userClient.getUser(userId, Duration.ofSeconds(5));
```

### 3. Thread Pool Starvation
```java
// All threads are busy waiting for external calls
@RestController
public class SlowController {
    @GetMapping("/slow")
    public String slow() {
        // Blocks the request thread for 30 seconds
        Thread.sleep(30000);
        return "Done";
    }
}
```

### 4. Lock Contention
```java
// Synchronized method blocks all threads
public synchronized void processOrder(Order order) {
    // Long-running operation
    externalService.call();  // Takes 5 seconds
}
```

### 5. I/O Bottlenecks
```java
// Reading large files synchronously
@GetMapping("/report")
public Report generateReport() {
    // Blocks the thread while reading a 1GB file
    List<Data> data = fileReader.read("/data/large-file.csv");
    return process(data);
}
```

## Diagnostic Commands

```bash
# Thread dump
jstack <pid>

# GC activity
jstat -gc <pid> 1s

# Heap usage
jmap -heap <pid>

# Native memory
jcmd <pid> VM.native_memory summary

# File descriptors
lsof -p <pid> | wc -l

# Network connections
ss -tuln | grep <port>
netstat -an | grep ESTABLISHED | wc -l
```

## Key Takeaway

> When CPU is low but requests are timing out, the bottleneck is likely **I/O** (database, external services, disk), **lock contention**, **thread pool exhaustion**, or **resource limits** (file descriptors, connections). Start by taking **thread dumps** to see what threads are waiting for, then check **connection pools**, **external service latency**, and **resource limits**. The key is to identify what the threads are **waiting for**, not what they're **computing**.
