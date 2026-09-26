# Users Are Seeing Stale Data Even Though the Database Has Been Updated. How Would You Debug It?

## The Problem

Users see outdated data even though the database has been updated. This is a **cache consistency** problem — the cache is serving stale data.

## Common Causes

### 1. Cache Without Invalidation

```java
@Service
public class UserService {
    
    @Cacheable("users")
    public User getUser(String userId) {
        return userRepository.findById(userId);
    }
    
    public void updateUser(User user) {
        userRepository.save(user);
        // BUG: Cache is not invalidated!
        // Next getUser() call returns stale data from cache
    }
}
```

**Fix**: Invalidate cache on update:

```java
@CacheEvict(value = "users", key = "#user.id")
public void updateUser(User user) {
    userRepository.save(user);
}
```

### 2. Cache TTL Too Long

```yaml
# Cache expires after 1 hour — too long!
spring:
  cache:
    type: redis
    redis:
      time-to-live: 3600000  # 1 hour
```

**Fix**: Reduce TTL or use write-through caching:

```yaml
spring:
  cache:
    redis:
      time-to-live: 300000  # 5 minutes
```

### 3. Read Replica Lag

```java
// Reading from a replica that hasn't caught up yet
public User getUser(String userId) {
    return replicaRepository.findById(userId);  // Stale data!
}
```

**Fix**: Read from primary for critical reads, or use read-after-write consistency:

```java
public User getUser(String userId) {
    // Read from primary for consistency
    return primaryRepository.findById(userId);
}
```

### 4. CDN Caching

```java
// CDN caches the response for too long
@GetMapping("/api/users/{id}")
@CacheControl(maxAge = 3600)  // 1 hour — too long!
public User getUser(@PathVariable String id) {
    return userService.getUser(id);
}
```

**Fix**: Use shorter cache times or cache invalidation:

```java
@GetMapping("/api/users/{id}")
@CacheControl(maxAge = 60)  // 1 minute
public User getUser(@PathVariable String id) {
    return userService.getUser(id);
}
```

### 5. Browser Caching

```java
// Browser caches the response
@GetMapping("/api/users/{id}")
public User getUser(@PathVariable String id) {
    return userService.getUser(id);
}
```

**Fix**: Add cache headers:

```java
@GetMapping("/api/users/{id}")
public ResponseEntity<User> getUser(@PathVariable String id) {
    User user = userService.getUser(id);
    return ResponseEntity.ok()
        .header("Cache-Control", "no-cache, no-store, must-revalidate")
        .body(user);
}
```

### 6. Cache-Aside Pattern Race Condition

```java
// Race condition: cache miss → DB read → cache write
// Another request updates the DB between cache miss and cache write
public User getUser(String userId) {
    User user = cache.get(userId);
    if (user == null) {
        user = db.get(userId);  // DB has new data
        cache.put(userId, user);  // But cache might have been updated by another thread
    }
    return user;
}
```

## Debugging Steps

### 1. Check Cache Hit/Miss Rates

```java
// Monitor cache metrics
@Component
public class CacheMonitor {
    
    @Autowired
    private CacheManager cacheManager;
    
    @Scheduled(fixedRate = 30000)
    public void logCacheStats() {
        Cache cache = cacheManager.getCache("users");
        if (cache instanceof RedisCache) {
            RedisTemplate<String, Object> template = ...;
            // Get cache stats
            Properties info = template.getConnectionFactory()
                .getConnection().info();
            System.out.println("Cache hits: " + info.getProperty("keyspace_hits"));
            System.out.println("Cache misses: " + info.getProperty("keyspace_misses"));
        }
    }
}
```

### 2. Check Cache Configuration

```bash
# Check Redis cache
redis-cli TTL user:123
redis-cli GET user:123

# Check cache entries
redis-cli KEYS "user:*"
```

### 3. Add Cache Logging

```java
@Cacheable(value = "users", key = "#userId")
public User getUser(String userId) {
    log.info("Cache miss for user: {}", userId);
    return userRepository.findById(userId);
}
```

### 4. Check Read Replica Lag

```sql
-- PostgreSQL
SELECT client_addr, state, sync_state, pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn) AS lag_bytes
FROM pg_stat_replication;

-- MySQL
SHOW SLAVE STATUS\G
-- Check Seconds_Behind_Master
```

### 5. Trace the Request

```java
// Use distributed tracing to see where data comes from
@Traced
public User getUser(String userId) {
    Span span = Span.current();
    span.setAttribute("cache.hit", cacheHit);
    span.setAttribute("source", cacheHit ? "cache" : "database");
    return user;
}
```

## Solutions

### 1. Cache Invalidation on Write

```java
@Service
@Transactional
public class UserService {
    
    @CacheEvict(value = "users", key = "#user.id")
    @CacheEvict(value = "user-names", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }
    
    @Cacheable("users")
    public User getUser(String userId) {
        return userRepository.findById(userId);
    }
}
```

### 2. Write-Through Caching

```java
@Service
public class UserService {
    
    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        User saved = userRepository.save(user);
        // Cache is updated automatically
        return saved;
    }
    
    @Cacheable("users")
    public User getUser(String userId) {
        return userRepository.findById(userId);
    }
}
```

### 3. Cache Versioning

```java
@Service
public class UserService {
    
    @Cacheable(value = "users", key = "{#userId, #version}")
    public User getUser(String userId, long version) {
        return userRepository.findById(userId);
    }
    
    public User updateUser(User user) {
        User saved = userRepository.save(user);
        // Increment version to invalidate old cache entries
        cacheManager.getCache("users").evict(
            new SimpleKey(saved.getId(), saved.getVersion() - 1));
        return saved;
    }
}
```

### 4. Read-Through with Consistency Check

```java
@Service
public class UserService {
    
    public User getUser(String userId) {
        User cached = cache.get(userId);
        if (cached != null) {
            // Check if cache is stale
            long dbVersion = userRepository.getVersion(userId);
            if (cached.getVersion() < dbVersion) {
                // Cache is stale — refresh
                cache.evict(userId);
                return getUser(userId);  // Recursive call with cache miss
            }
            return cached;
        }
        
        User user = userRepository.findById(userId);
        cache.put(userId, user);
        return user;
    }
}
```

### 5. Event-Driven Cache Invalidation

```java
@Component
public class CacheInvalidationListener {
    
    @EventListener
    public void handleUserUpdated(UserUpdatedEvent event) {
        cacheManager.getCache("users").evict(event.getUserId());
    }
}
```

## Key Takeaway

> Stale data is typically caused by **cache invalidation issues** — either the cache isn't being invalidated on updates, the TTL is too long, or there's a race condition. Debug by checking cache hit/miss rates, cache configuration, read replica lag, and using distributed tracing. Fix with proper cache invalidation strategies: `@CacheEvict` on writes, write-through caching, cache versioning, or event-driven invalidation.

---

## Production Topic: Caching Strategy (Redis) — Three Strategies, Three Failure Modes

### The Three Cache Strategies

#### 1. Cache-Aside (Lazy Loading)

```java
// Read: Check cache first, then DB
public User getUser(String userId) {
    User user = cache.get(userId);
    if (user == null) {
        user = userRepository.findById(userId);
        cache.put(userId, user);
    }
    return user;
}

// Write: Update DB, then invalidate cache
public void updateUser(User user) {
    userRepository.save(user);
    cache.evict(user.getId());  // Invalidate cache
}
```

**Pros**:
- Simple to implement
- Cache only stores requested data
- DB is source of truth

**Cons**:
- **Race condition**: Cache miss → DB read → cache write (another request updates DB between miss and write)
- **Stale data**: If cache eviction fails, stale data persists
- **Cache penetration**: Requests for non-existent data hit DB every time

**Failure Mode**: Stale data under load if eviction fails or TTL is too long.

#### 2. Write-Through

```java
// Write: Update cache AND DB simultaneously
public void updateUser(User user) {
    // Write to cache first
    cache.put(user.getId(), user);
    // Then write to DB
    userRepository.save(user);
}

// Read: Always from cache
public User getUser(String userId) {
    return cache.get(userId);  // Cache is always up-to-date
}
```

**Pros**:
- Cache is always up-to-date
- No cache invalidation needed
- Strong consistency

**Cons**:
- Write latency increases (must write to both)
- Cache stores data that might never be read
- More complex to implement

**Failure Mode**: If cache write fails but DB write succeeds, cache and DB are inconsistent.

#### 3. Write-Behind (Write-Back)

```java
// Write: Update cache immediately, DB asynchronously
public void updateUser(User user) {
    cache.put(user.getId(), user);
    // DB update happens asynchronously in background
    asyncDbWriter.queue(user);
}

// Background worker writes to DB
@Scheduled(fixedRate = 1000)
public void flushCacheToDb() {
    while (!writeQueue.isEmpty()) {
        User user = writeQueue.poll();
        userRepository.save(user);
    }
}
```

**Pros**:
- Lowest write latency
- Can batch DB writes
- Absorbs write spikes

**Cons**:
- **Data loss risk**: If cache fails before DB write, data is lost
- **Complexity**: Need to handle crash recovery
- **Eventual consistency**: DB lags behind cache

**Failure Mode**: Data loss on cache failure. Most dangerous strategy.

### Choosing an Eviction Policy

```java
// LRU (Least Recently Used) — default, good for most cases
cache.evict(userId);  // Evicts least recently used entry

// LFU (Least Frequently Used) — good for hot data
// Keeps frequently accessed data longer

// TTL (Time To Live) — good for time-sensitive data
cache.put(userId, user, Duration.ofMinutes(5));  // Expires after 5 minutes

// FIFO (First In First Out) — simple, but not optimal
```

**Wrong eviction policy → stale data under load**:
- TTL too long → stale data persists
- No eviction → cache grows indefinitely
- LRU not always right — if access pattern changes, hot data gets evicted

### Key Takeaways

- **Cache-aside**: Simple, but has race conditions. Good for most cases.
- **Write-through**: Strong consistency, but higher write latency. Good for financial data.
- **Write-behind**: Lowest latency, but risk of data loss. Good for analytics/logging.
- **LRU is not always right** — choose eviction policy based on access patterns.
- **Wrong eviction policy → stale data under load** — always test with production-like traffic.
