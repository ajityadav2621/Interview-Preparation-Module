# Your API Suddenly Becomes Slow. How Do You Identify the Bottleneck?

## The Investigation Approach

When an API suddenly becomes slow, you need a systematic approach to identify the bottleneck. Here's the step-by-step process:

## Step 1: Confirm the Problem

### Check Metrics
- **Response time**: Is it consistently slow or intermittent?
- **Throughput**: Are requests per second dropping?
- **Error rate**: Are there more errors?
- **Resource usage**: CPU, memory, disk I/O, network

### Check Recent Changes
- **Deployments**: Was there a recent deployment?
- **Traffic spikes**: Is traffic higher than usual?
- **Data growth**: Has the dataset grown significantly?
- **External dependencies**: Are downstream services slower?

## Step 2: Check Application-Level Metrics

### JVM Metrics
```bash
# Check GC activity
jstat -gc <pid> 1s

# Check thread dumps
jstack <pid> > threaddump.txt

# Check heap usage
jmap -heap <pid>
```

**Look for**:
- Frequent Full GC (memory pressure)
- Thread contention (BLOCKED threads)
- High CPU usage (busy loops)

### Application Logs
```bash
# Check for errors, warnings, slow queries
grep -i "error\|warn\|slow" application.log

# Check for specific patterns
grep "slow query" application.log | tail -20
```

## Step 3: Check Database Performance

### Database Query Analysis
```sql
-- Check slow queries
SHOW PROCESSLIST;  -- MySQL
SELECT * FROM pg_stat_activity WHERE state = 'active';  -- PostgreSQL

-- Check query execution plans
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;
```

### Database Metrics
- **Connection pool usage**: Are connections exhausted?
- **Query latency**: Are specific queries slow?
- **Lock wait time**: Are there lock contention issues?
- **CPU and I/O**: Is the database server overloaded?

### Connection Pool Monitoring
```java
// HikariCP metrics
HikariDataSource ds = (HikariDataSource) dataSource;
System.out.println("Active connections: " + ds.getHikariPoolMXBean().getActiveConnections());
System.out.println("Idle connections: " + ds.getHikariPoolMXBean().getIdleConnections());
System.out.println("Total connections: " + ds.getHikariPoolMXBean().getTotalConnections());
```

## Step 4: Check External Dependencies

### HTTP Client Metrics
```java
// Monitor external API calls
@RestController
public class OrderController {
    @Timed("order.process")
    public ResponseEntity<Order> processOrder(@RequestBody OrderRequest request) {
        // External calls will be timed automatically
        User user = userService.getUser(request.getUserId());
        Payment payment = paymentService.process(request.getAmount());
        return ResponseEntity.ok(new Order(user, payment));
    }
}
```

### Check Network Latency
```bash
# Ping external services
ping api.external-service.com

# Check DNS resolution time
dig api.external-service.com

# Check network throughput
iperf3 -c external-service-host
```

## Step 5: Check Infrastructure

### Container/VM Metrics
```bash
# CPU usage
top -p <pid>

# Memory usage
free -m

# Disk I/O
iostat -x 1

# Network
iftop
ss -tuln  # Check open ports and connections
```

### Load Balancer Metrics
- **Request rate**: Is traffic higher than expected?
- **Error rate**: Are there 5xx errors?
- **Latency**: Is the load balancer adding latency?
- **Backend health**: Are backend instances healthy?

## Step 6: Use Distributed Tracing

### OpenTelemetry / Jaeger
```java
// Add tracing to your service
@Traced
public Order processOrder(OrderRequest request) {
    Span span = Span.current();
    span.setAttribute("order.id", request.getOrderId());
    
    User user = userService.getUser(request.getUserId());  // Traced
    Payment payment = paymentService.process(request.getAmount());  // Traced
    
    return new Order(user, payment);
}
```

### Trace Analysis
- **Identify slow spans**: Which service/operation is taking the most time?
- **Check for errors**: Are there failed spans?
- **Analyze dependencies**: Which external calls are slow?

## Step 7: Profiling

### CPU Profiling
```bash
# Use jcmd for sampling
jcmd <pid> VM.native_memory summary

# Or use async-profiler
./profiler.sh -e cpu -d 30 -f profile.html <pid>
```

### Flame Graphs
```bash
# Generate flame graph
./profiler.sh -e cpu -d 30 -f profile.html <pid>
# Open profile.html in browser to see CPU hotspots
```

## Diagnostic Checklist

| Check | Tool | What to Look For |
|-------|------|-----------------|
| GC activity | `jstat`, GC logs | Frequent Full GC, long pauses |
| Thread dumps | `jstack` | Deadlocks, thread contention |
| Heap usage | `jmap`, `jstat` | Memory leaks, high utilization |
| Database queries | `EXPLAIN`, slow query log | Slow queries, missing indexes |
| Connection pools | JMX, logs | Exhausted pools, connection leaks |
| External APIs | Tracing, logs | Slow external calls |
| CPU usage | `top`, `htop` | High CPU, busy loops |
| Network | `iftop`, `ping` | Network latency, packet loss |
| Disk I/O | `iostat` | High I/O wait, disk saturation |
| Logs | `grep`, ELK | Errors, warnings, slow operations |

## Common Root Causes

1. **Database issues**: Slow queries, missing indexes, connection pool exhaustion
2. **GC pressure**: Memory leaks, insufficient heap, frequent Full GC
3. **Thread contention**: Deadlocks, lock contention, thread pool exhaustion
4. **External service latency**: Downstream API calls, network issues
5. **Resource exhaustion**: CPU, memory, disk, or network saturation
6. **Configuration changes**: New deployment with different settings
7. **Data growth**: Queries that were fast with small data are now slow
8. **Traffic spikes**: Unexpected increase in request volume

## Key Takeaway

> When an API becomes slow, follow a systematic approach: **confirm the problem** → **check application metrics** → **check database** → **check external dependencies** → **check infrastructure** → **use distributed tracing** → **profile if needed**. The key is to narrow down the scope quickly using metrics and tracing, rather than guessing.
