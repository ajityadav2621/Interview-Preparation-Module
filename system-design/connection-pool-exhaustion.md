# A Database Connection Pool Is Exhausted While the Database Itself Is Healthy. Why?

## The Problem

The database is healthy and responsive, but the application can't get database connections. All requests are timing out or failing with "Connection is not available" errors.

## Root Causes

### 1. Connection Leaks

Connections are not being returned to the pool:

```java
// BAD: Connection not closed
public User getUser(String id) {
    Connection conn = dataSource.getConnection();
    // Process query...
    // Missing: conn.close() — connection leaks!
    return user;
}

// GOOD: Use try-with-resources
public User getUser(String id) {
    try (Connection conn = dataSource.getConnection()) {
        // Process query...
        return user;
    }  // Connection automatically returned
}
```

### 2. Long-Running Transactions

Connections are held for too long:

```java
@Transactional
public void processBatch(List<Order> orders) {
    for (Order order : orders) {
        // Each operation holds the connection
        processOrder(order);  // Takes 5 seconds each
    }
    // Connection held for 5 * orders.size() seconds!
}
```

### 3. Insufficient Pool Size

The pool is too small for the workload:

```yaml
# Pool too small for 100 concurrent requests
spring:
  datasource:
    hikari:
      maximum-pool-size: 10  # Only 10 connections!
```

### 4. Connection Timeout Too Short

```yaml
# Timeout too short — connections time out before they can be used
spring:
  datasource:
    hikari:
      connection-timeout: 100  # 100ms — too short!
```

### 5. Idle Connection Timeout Issues

```yaml
# Idle connections are closed too quickly
spring:
  datasource:
    hikari:
      idle-timeout: 1000  # 1 second — too aggressive
      max-lifetime: 30000  # 30 seconds — too short
```

### 6. Database-Side Connection Limits

The database has a lower connection limit than the pool:

```sql
-- PostgreSQL
SHOW max_connections;  -- e.g., 100

-- MySQL
SHOW VARIABLES LIKE 'max_connections';  -- e.g., 151
```

If the pool tries to create more connections than the database allows, connections will fail.

### 7. Network Issues

Network partitions or firewall rules can cause connections to hang:

```java
// Connection hangs indefinitely
Connection conn = dataSource.getConnection();  // Never returns
```

## How to Diagnose

### 1. Check Pool Metrics

```java
HikariDataSource ds = (HikariDataSource) dataSource;
HikariPoolMXBean poolBean = ds.getHikariPoolMXBean();

System.out.println("Active connections: " + poolBean.getActiveConnections());
System.out.println("Idle connections: " + poolBean.getIdleConnections());
System.out.println("Total connections: " + poolBean.getTotalConnections());
System.out.println("Threads awaiting connection: " + poolBean.getThreadsAwaitingConnection());
```

### 2. Check for Connection Leaks

```java
// Enable leak detection
HikariConfig config = new HikariConfig();
config.setLeakDetectionThreshold(60000);  // 60 seconds
```

This will log a warning if a connection is held for more than 60 seconds.

### 3. Monitor with JMX

```bash
# Use jconsole or jmc to monitor HikariCP metrics
# Look for:
# - ActiveConnections
# - IdleConnections
# - TotalConnections
# - ThreadsAwaitingConnection
# - ConnectionTimeouts
```

### 4. Check Database Side

```sql
-- PostgreSQL
SELECT count(*) FROM pg_stat_activity WHERE datname = 'mydb';

-- MySQL
SHOW PROCESSLIST;
SELECT COUNT(*) FROM INFORMATION_SCHEMA.PROCESSLIST;
```

## Solutions

### 1. Fix Connection Leaks

```java
// Always use try-with-resources
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement(sql)) {
    // Process query
}

// Or use Spring's @Transactional which manages connections automatically
@Transactional
public void processOrder(Order order) {
    // Connection is automatically managed
    orderRepository.save(order);
}
```

### 2. Optimize Pool Configuration

```yaml
spring:
  datasource:
    hikari:
      # Pool size: 2 * CPU cores + effective spindle count
      maximum-pool-size: 20
      minimum-idle: 5
      
      # Connection timeout: how long to wait for a connection
      connection-timeout: 30000  # 30 seconds
      
      # Idle timeout: how long to keep idle connections
      idle-timeout: 600000  # 10 minutes
      
      # Max lifetime: how long a connection can live
      max-lifetime: 1800000  # 30 minutes
      
      # Leak detection
      leak-detection-threshold: 60000  # 60 seconds
```

### 3. Use Connection Pool Per Service

```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @Primary
    public DataSource primaryDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://primary:5432/mydb");
        config.setMaximumPoolSize(15);
        return new HikariDataSource(config);
    }
    
    @Bean
    @Secondary
    public DataSource replicaDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://replica:5432/mydb");
        config.setMaximumPoolSize(10);
        return new HikariDataSource(config);
    }
}
```

### 4. Optimize Long-Running Transactions

```java
// BAD: Long transaction
@Transactional
public void processBatch(List<Order> orders) {
    for (Order order : orders) {
        processOrder(order);  // Each takes time
    }
}

// GOOD: Process in smaller batches
public void processBatch(List<Order> orders) {
    List<List<Order>> batches = Lists.partition(orders, 10);
    for (List<Order> batch : batches) {
        processBatchChunk(batch);
    }
}

@Transactional
public void processBatchChunk(List<Order> batch) {
    for (Order order : batch) {
        processOrder(order);
    }
}
```

### 5. Use Read Replicas

```java
// Route read queries to replicas
@Transactional(readOnly = true)
public List<Order> getOrders(String userId) {
    // Uses read replica connection pool
    return orderRepository.findByUserId(userId);
}

@Transactional
public void createOrder(Order order) {
    // Uses primary connection pool
    orderRepository.save(order);
}
```

### 6. Connection Pool Monitoring

```java
@Component
public class ConnectionPoolMonitor {
    
    @Autowired
    private DataSource dataSource;
    
    @Scheduled(fixedRate = 30000)
    public void monitorPool() {
        if (dataSource instanceof HikariDataSource) {
            HikariPoolMXBean pool = ((HikariDataSource) dataSource).getHikariPoolMXBean();
            
            int active = pool.getActiveConnections();
            int idle = pool.getIdleConnections();
            int total = pool.getTotalConnections();
            int waiting = pool.getThreadsAwaitingConnection();
            
            if (waiting > 0) {
                log.warn("Connection pool exhausted: {} threads waiting", waiting);
            }
            
            double usage = (double) active / total;
            if (usage > 0.8) {
                log.warn("Connection pool usage high: {}%", (int)(usage * 100));
            }
        }
    }
}
```

## Key Takeaway

> Connection pool exhaustion while the database is healthy is typically caused by **connection leaks** (connections not returned to the pool), **long-running transactions** (connections held too long), or **insufficient pool size**. Diagnose by checking pool metrics, enabling leak detection, and monitoring database-side connections. Fix by using try-with-resources, optimizing transaction boundaries, and properly sizing the connection pool.

---

## Production Topic: Connection Pooling (HikariCP) — The Right Pool Size

### The Default Pool Size Is Almost Always Wrong

Spring Boot's default HikariCP pool size is 10. This is almost never the right size for production.

### The Formula

```yaml
# Baseline formula for pool size:
# Pool size = (CPU cores × 2) + effective spindle count

# Example: 4 CPU cores, SSD (1 spindle)
# Pool size = (4 × 2) + 1 = 9

# Example: 8 CPU cores, HDD (4 spindles)
# Pool size = (8 × 2) + 4 = 20
```

**Why this formula?**
- Each connection can execute one query at a time
- CPU cores determine how many queries can execute concurrently
- Disk spindles determine I/O parallelism (HDD = multiple spindles, SSD = 1)

### Too Many Connections → Database Dies

```yaml
# BAD: Pool too large
spring:
  datasource:
    hikari:
      maximum-pool-size: 100  # Way too many!

# What happens:
# 1. Database receives 100 concurrent queries
# 2. CPU/IO on DB spikes to 100%
# 3. Query response times increase
# 4. Connection context switching overhead
# 5. Database becomes unresponsive
```

### Too Few Connections → Threads Wait at 3am

```yaml
# BAD: Pool too small
spring:
  datasource:
    hikari:
      maximum-pool-size: 2  # Way too few!

# What happens:
# 1. Only 2 queries can run concurrently
# 2. Other threads wait for connections
# 3. Response time increases dramatically
# 4. Thread pool exhaustion
# 5. Users see timeouts
```

### Correct Configuration

```yaml
spring:
  datasource:
    hikari:
      # Pool size: calculated based on hardware
      maximum-pool-size: 20
      minimum-idle: 5
      
      # Connection timeout: how long to wait for a connection
      connection-timeout: 30000  # 30 seconds
      
      # Idle timeout: how long to keep idle connections
      idle-timeout: 600000  # 10 minutes
      
      # Max lifetime: how long a connection can live
      max-lifetime: 1800000  # 30 minutes
      
      # Leak detection
      leak-detection-threshold: 60000  # 60 seconds
```

### Monitoring Pool Metrics

```java
@Component
public class ConnectionPoolMonitor {
    
    @Autowired
    private DataSource dataSource;
    
    @Scheduled(fixedRate = 30000)
    public void monitorPool() {
        if (dataSource instanceof HikariDataSource) {
            HikariPoolMXBean pool = ((HikariDataSource) dataSource).getHikariPoolMXBean();
            
            int active = pool.getActiveConnections();
            int idle = pool.getIdleConnections();
            int total = pool.getTotalConnections();
            int waiting = pool.getThreadsAwaitingConnection();
            
            // Alert if threads are waiting
            if (waiting > 0) {
                log.warn("Connection pool exhausted: {} threads waiting", waiting);
            }
            
            // Alert if pool usage is high
            double usage = (double) active / total;
            if (usage > 0.8) {
                log.warn("Connection pool usage high: {}%", (int)(usage * 100));
            }
            
            log.info("Pool: active={}, idle={}, total={}, waiting={}", 
                active, idle, total, waiting);
        }
    }
}
```

### Key Takeaway

> The default pool size of 10 is almost always wrong. Use the formula `(CPU cores × 2) + disk spindles` as a baseline. Too many connections kill the database; too few cause threads to wait. Monitor pool metrics and adjust based on actual usage patterns. The right pool size depends on your hardware, workload, and database capacity.
