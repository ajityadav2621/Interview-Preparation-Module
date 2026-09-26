# Incident Response, Idempotency, Messaging & Distributed Systems

## 9. How Would You Make a REST API Idempotent?

Idempotency means that **multiple identical requests have the same effect as a single request**.

### Strategy 1: Idempotency Key (Header)
```java
@RestController
public class PaymentController {
    
    @PostMapping("/payments")
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody PaymentRequest request) {
        
        return idempotencyService.executeIdempotent(idempotencyKey, request, () -> {
            return paymentService.process(request);
        });
    }
}

@Service
public class IdempotencyService {
    
    private final RedisTemplate<String, StoredResponse> redis;
    
    // Required Redis serializer configuration (otherwise raw bytes / type errors)
    @Bean
    public RedisTemplate<String, StoredResponse> idempotencyRedisTemplate(
            RedisConnectionFactory cf) {
        RedisTemplate<String, StoredResponse> tpl = new RedisTemplate<>();
        tpl.setConnectionFactory(cf);
        Jackson2JsonRedisSerializer<StoredResponse> ser =
            new Jackson2JsonRedisSerializer<>(StoredResponse.class);
        tpl.setKeySerializer(RedisSerializer.string());
        tpl.setValueSerializer(ser);
        tpl.afterPropertiesSet();
        return tpl;
    }
    
    public <T> ResponseEntity<T> executeIdempotent(
            String key, Object request, Supplier<ResponseEntity<T>> action) {
        
        String fingerprint = fingerprint(request);
        String cacheKey = key + ":" + fingerprint;
        
        // Check existing result
        CachedResult<T> existing = redis.get(cacheKey);
        if (existing != null) {
            if (existing.isCompleted()) {
                return ResponseEntity.status(existing.getStatus()).body(existing.getResult());
            }
            // In-flight — wait or return 409
            throw new ConcurrentRequestException("Request already in progress");
        }
        
        // Reserve the key with NX
        ReservationToken token = redis.setIfAbsent(cacheKey, "PENDING", 24, HOURS);
        if (token == null) {
            throw new ConcurrentRequestException("Duplicate request");
        }
        
        try {
            ResponseEntity<T> response = action.get();
            redis.set(cacheKey, new CachedResult<>(response.getStatusCode(), response.getBody()), 24, HOURS);
            return response;
        } catch (Exception e) {
            redis.delete(cacheKey);   // Allow retry
            throw e;
        }
    }
}
```

### Strategy 2: Database-Level Idempotency
```sql
CREATE TABLE payments (
    id          BIGINT PRIMARY KEY,
    request_id  VARCHAR(64) UNIQUE NOT NULL,  -- Client-supplied
    amount      DECIMAL(19,2),
    status      VARCHAR(20),
    created_at  TIMESTAMP
);
```

### Strategy 3: Natural Idempotency
- `PUT /users/{id}` — replacing entire resource is naturally idempotent
- `DELETE /users/{id}` — deleting twice is safe

---

## 10. What Happens When the Same Request Is Processed Twice?

### Without Idempotency:
- **Duplicate charge** — customer charged twice
- **Duplicate order** — two orders created
- **Incorrect inventory** — items reserved twice
- **Inconsistent state** — balances off

### With Idempotency Key:
```
1st request  → processed, response cached with key K
2nd request  → returns cached response (no side-effect)
3rd request  → returns cached response
```

### Common Patterns:
- **Idempotency Key** header (Stripe-style)
- **Request fingerprint** (hash of payload)
- **Database unique constraint** (request_id)
- **Optimistic locking** with version field
- **Compare-and-swap** with conditional update

---

## 11. How Do You Handle Duplicate Messages?

In messaging systems (Kafka, RabbitMQ), duplicates can occur due to **at-least-once delivery**.

### Strategy 1: Idempotent Consumer
```java
@KafkaListener(topics = "orders")
public void consume(OrderEvent event) {
    // Dedup by business key
    if (processedEventRepository.existsByEventId(event.getEventId())) {
        log.info("Duplicate event skipped: {}", event.getEventId());
        return;
    }
    
    processOrder(event);
    
    // Mark as processed (must be atomic with business logic)
    processedEventRepository.save(new ProcessedEvent(event.getEventId()));
}
```

### Strategy 2: Database Unique Constraint
```java
@Transactional
public void processEvent(OrderEvent event) {
    // Insert idempotency record first — fails on duplicate
    idempotencyRepository.save(new IdempotencyRecord(event.getEventId()));
    
    // Now do business work
    orderService.createOrder(event);
}
```

### Strategy 3: Kafka Exactly-Once (Idempotent Producer + Transactions)
```java
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "tx-1");

// Consumer-side: store offset in the same DB transaction as business work
props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
```

### Strategy 4: Deduplication Window in Cache
```java
public void handleMessage(Message msg) {
    String dedupKey = msg.getMessageId();
    Boolean firstTime = redis.setIfAbsent("dedup:" + dedupKey, "1", Duration.ofHours(24));
    if (Boolean.FALSE.equals(firstTime)) {
        return;  // already processed
    }
    process(msg);
}
```

---

## 12. What Happens When a Kafka Consumer Crashes While Processing a Message?

### Default Behavior (Auto-Commit):
- Offset already committed BEFORE processing → **message is lost**
- Offset committed AFTER processing → message **re-delivered** on restart (duplicate)

### Safe Pattern: Manual Commit After Success
```java
@KafkaListener(topics = "orders", containerFactory = "manualAckFactory")
public void consume(ConsumerRecord<String, OrderEvent> record, Acknowledgment ack) {
    try {
        orderService.process(record.value());
        ack.acknowledge();   // commit only after success
    } catch (Exception e) {
        // don't ack — message will be re-delivered
        log.error("Processing failed", e);
    }
}
```

### Even Safer: Idempotent Processing + Transactional Outbox (Spring Boot 3+)
> **Note:** `ChainedTransactionManager` was deprecated and removed from Spring Framework 6+ / Spring Boot 3+. The modern equivalent is the **Transactional Outbox Pattern**: write business data + outbox event in the same DB transaction, then a separate worker publishes events to Kafka.

```java
@Transactional
@KafkaListener(topics = "orders")
public void consume(OrderEvent event) {
    // 1. Insert idempotency record
    // 2. Business logic
    // 3. Outbox row written in the SAME transaction → published asynchronously
    outboxRepository.save(new OutboxEntry(event.getEventId(), serialize(event)));
    // When transaction commits, the outbox row is durable and a poller
    // (or Debezium CDC) publishes it to Kafka with the consumer's offset
}
```

### Crash Recovery Flow:
1. Consumer crashes mid-processing
2. New consumer (or restarted consumer) takes over partition
3. Reads from last **committed** offset
4. Re-processes any uncommitted messages
5. Idempotency keys prevent duplicate side-effects

---

## 13. How Do You Handle Poison Messages?

**Poison message** = a message that consistently fails processing (bad data, schema mismatch, unhandled exception).

### Detection:
```java
int maxAttempts = 3;
if (event.getRetryCount() >= maxAttempts) {
    deadLetterPublisher.publish(event);
    return; // skip
}
```

### Strategy 1: Dead Letter Queue (DLQ)
```java
@RetryableTopic(
    attempts = "3",
    backoff = @Backoff(delay = 1000, multiplier = 2),
    dltStrategy = DltStrategy.FAIL_ON_ERROR
)
@KafkaListener(topics = "orders")
public void process(OrderEvent event) {
    orderService.process(event);
}

@KafkaListener(topics = "orders-dlt")
public void handleDlt(OrderEvent event) {
    alertService.send("Poison message received: " + event.getEventId());
    poisonRepository.save(event);
}
```

### Strategy 2: Retry Topic + Parking Lot
```
orders        → orders-retry-0 (1s delay)
orders-retry-0 → orders-retry-1 (10s delay)
orders-retry-1 → orders-retry-2 (60s delay)
orders-retry-2 → orders-parking-lot   (manual intervention)
```

### Strategy 3: Quarantine + Alert
- Move message to quarantine topic
- Page on-call engineer
- Provide tooling to inspect & replay

---

## 14. What Is a Dead Letter Queue?

A **DLQ** is a queue/topic that receives messages that cannot be processed successfully.

### Purpose:
- **Prevent infinite retry loops**
- **Preserve messages** for forensic analysis
- **Allow manual replay** after fixing the issue
- **Decouple** failed processing from main flow

### Implementation (Spring Kafka):
```yaml
spring:
  kafka:
    listener:
      dead-letter-publishing-reporter:
        enabled: true
```

```java
@RetryableTopic(
    attempts = "4",
    dltStrategy = DltStrategy.FAIL_ON_ERROR
)
@KafkaListener(topics = "payments")
public void process(PaymentEvent event) { ... }

// Separate consumer for DLT
@KafkaListener(topics = "payments-dlt")
public void dlt(PaymentEvent event) {
    log.error("Payment DLT: {}", event);
    metrics.incrementCounter("payment.dlt");
}
```

### Best Practices:
- Set **TTL** on DLT (don't accumulate forever)
- **Alert** when DLT depth grows
- Provide a **replay tool** to drain DLT back to main topic
- **Tag** messages with failure reason + retry count

---

## 15. How Do You Handle Message Ordering?

### The Problem:
In Kafka, ordering is **guaranteed only within a partition**. Multiple consumers / partitions = no global order.

### Strategy 1: Partition by Key (Same Key → Same Partition)
```java
// Orders for same customer go to same partition → ordered
producer.send(new ProducerRecord<>("orders", order.getCustomerId(), order));
```

### Strategy 2: Single Partition (Use Sparingly)
- Only for low-throughput scenarios where global order matters
- Bottleneck — single consumer

### Strategy 3: Sequence Numbers + Buffering
```java
class OrderEvent {
    private String customerId;
    private long sequenceNumber;   // monotonic per partition
}

public void process(OrderEvent event) {
    bufferByCustomer(event.getCustomerId()).add(event);
    flushIfReady(event.getCustomerId());
}

void flushIfReady(String customerId) {
    long expected = lastProcessedSeq.get(customerId) + 1;
    OrderEvent next = buffer.get(customerId).poll(expected);
    if (next != null) {
        applyEvent(next);
        lastProcessedSeq.put(customerId, next.getSequenceNumber());
    }
}
```

### Strategy 4: Application-Level Sequencing
- Use **timestamps** + **out-of-order tolerance** window
- Allow events to arrive slightly out of order
- Resolve conflicts using **last-write-wins** or **CRDT**

---

## 16. What Happens When a Database Becomes Unavailable?

### Symptoms:
- **Connection refused / timeout** — TCP errors
- **Slow queries** — every request hangs
- **Connection pool exhaustion** — see Q17
- **Cascading failures** — calling services fail

### Defense Strategies:

**1. Connection Pool with Fast Failure**
```yaml
spring:
  datasource:
    hikari:
      connection-timeout: 3000        # fail fast
      validation-timeout: 1000
      maximum-pool-size: 20
      leak-detection-threshold: 30000
```

**2. Circuit Breaker on DB**
```java
@CircuitBreaker(name = "database", fallbackMethod = "dbFallback")
public Account getAccount(String id) {
    return jdbcTemplate.queryForObject(
        "SELECT * FROM accounts WHERE id = ?", 
        accountMapper, id);
}

public Account dbFallback(String id, Exception e) {
    return cacheService.getCachedAccount(id);   // degraded mode
}
```

**3. Read Replicas for Read Traffic**
```
Primary (writes) → Streaming Replication → Read Replica
```
Route reads to replica, writes to primary. Replica failure → serve from cache.

**4. Cache-First Pattern**
```java
public Account getAccount(String id) {
    return cache.get(id, Account.class)
        .orElseGet(() -> {
            Account a = db.findById(id);
            cache.put(id, a, 5, MINUTES);
            return a;
        });
}
```

**5. Fail-Write vs Fail-Read**
- **Reads** → return cached/stale data, mark as degraded
- **Writes** → reject with 503, queue for retry, or persist locally (outbox)

---

## 17. How Would You Handle Database Connection Pool Exhaustion?

### Why It Happens:
- Queries are slow → connections held longer
- Traffic spike → more concurrent requests
- Connection leak (not closed)
- Pool sized too small

### Detection:
```yaml
hikari:
  leak-detection-threshold: 30000  # log if connection held > 30s
```

```java
// Metrics to watch
hikaricp.connections.active
hikaricp.connections.idle
hikaricp.connections.pending    // waiting for connection
hikaricp.connections.timeout    // count of timeouts
```

### Solutions:

**1. Pool Sizing (Little's Law)**
```
pool_size = (core_count * 2) + effective_spindle_count
For SSD:  pool_size ≈ cores * 4
```

**2. Query Timeout**
```java
@QueryHints({@QueryHint(name = "jakarta.persistence.query.timeout", value = "3000")})
```

**3. Closing Connections Properly**
```java
try (Connection conn = dataSource.getConnection()) {   // try-with-resources
    // use connection
}  // auto-closed even on exception
```

**4. Statement Timeout**
```sql
SET STATEMENT max_statement_time = 5 FOR;   -- MySQL
```

**5. Bulkhead Between Read/Write**
```java
// Separate pools for read vs write
@Bean @Qualifier("read") DataSource readDataSource() { ... }
@Bean @Qualifier("write") DataSource writeDataSource() { ... }
```

**6. Graceful Degradation**
- Return cached data
- Reject with `503 Retry-After`
- Queue requests (if async OK)

---

## 18. What Happens When One Microservice Runs Out of Memory?

### Symptoms:
- `OutOfMemoryError` thrown
- **Full GC continuously** (GC death spiral)
- Container gets **OOMKilled** by Kubernetes
- Pod restarts → brief downtime

### Causes:
- **Memory leak** — references not released (e.g., static collections growing)
- **Cache with no eviction** — unbounded growth
- **Large object allocation** — big payloads, huge collections
- **ThreadLocal leak** — not removed
- **ClassLoader leak** — redeploys in app servers

### Detection:
```bash
# JVM flags for diagnosis
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heapdump.hprof
-XX:+ExitOnOutOfMemoryError   # fail fast
```

### Investigation:
```bash
# 1. Capture heap dump
jmap -dump:format=b,file=heap.hprof <pid>

# 2. Use Eclipse MAT / VisualVM
# Find "Leak Suspects" → largest retained objects

# 3. Inspect running histograms
jmap -histo:live <pid> | head -50
```

### Prevention:
```yaml
# Kubernetes — set memory limits
resources:
  limits:
    memory: "1Gi"
  requests:
    memory: "512Mi"
```

```java
// JVM settings
-Xms512m -Xmx1g
-XX:MaxRAMPercentage=75.0

// Use bounded caches
CacheBuilder.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(10, MINUTES)
    .build();
```

### Mitigation in Kubernetes:
- Set `restartPolicy: Always` (restart on OOM)
- **Horizontal Pod Autoscaler** for traffic spikes
- Use **HPA + memory metric** to scale before OOM
- **PDB** (Pod Disruption Budget) to avoid cascading restart

---

## 19. How Would You Troubleshoot High CPU in One Service?

### Step 1: Confirm It's Real
```bash
# Top processes
top -c

# JVM-specific
jps -l   # list JVM processes
```

### Step 2: Find the Hot Thread
```bash
# Send SIGQUIT to dump thread stacks
kill -3 <pid>

# Or use jstack
jstack <pid> > thread-dump.txt

# Identify threads consuming CPU
top -H -p <pid>   # show threads
printf "%x\n" <tid>   # convert to hex, search in thread-dump
```

### Step 3: Identify the Hotspot
```bash
# Async profiler (low overhead)
./asprof -e cpu -d 30 -f flamegraph.html <pid>

# Or JFR (JDK Flight Recorder)
jcmd <pid> JFR.start duration=60s filename=rec.jfr
```

### Common Root Causes:

| Symptom | Cause | Fix |
|---------|-------|-----|
| Single thread 100% CPU | Infinite loop / regex backtracking | Code review, add bounds |
| Many threads high CPU | Thread contention | Reduce lock scope, use CAS |
| GC threads high CPU | Frequent GC | Tune heap, fix leak |
| JIT compiler high CPU | Startup warmup | AOT, lazy compile |

### Step 4: Fix & Verify
- Code fix → deploy
- Configuration change → update deployment
- Capacity change → scale out

---

## 20. What Happens When a Kubernetes Pod Keeps Restarting?

### Pod Restart Loop (CrashLoopBackOff):
```
Pod starts → application crashes → kubelet restarts → crash again → backoff
```

### Diagnosis:
```bash
kubectl get pods                          # see status
kubectl describe pod <pod-name>           # events, exit code, restarts
kubectl logs <pod-name> --previous        # logs from last crashed container
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Common Causes:

| Exit Code | Cause |
|-----------|-------|
| 0 | Normal (but check why restart policy triggered) |
| 1 | Application error (config, missing dep) |
| 137 | OOMKilled (out of memory) |
| 139 | Segfault (native code issue) |
| 143 | SIGTERM (graceful shutdown too slow) |

### Solutions:

**1. Liveness Probe Failing**
- Probe endpoint is slow or returns 500
- Make probe lightweight (separate `/health/live`)

**2. App Not Ready in Time**
- Startup probe to give app time to warm up
```yaml
startupProbe:
  httpGet: { path: /health/startup, port: 8080 }
  failureThreshold: 30
  periodSeconds: 5
```

**3. OOMKilled**
```yaml
resources:
  limits:
    memory: "2Gi"
  requests:
    memory: "1Gi"
```

**4. Config / Secret Missing**
```bash
kubectl describe pod <pod-name>   # look for "MountVolume" or "CreateContainerConfigError"
```

**5. Database Unreachable on Startup**
- Add retry logic in `@PostConstruct`
- Use init containers to wait for dependencies

---

## 21. How Do Readiness and Liveness Probes Help?

### Liveness Probe — "Should I restart this container?"
- **Purpose**: Detect deadlocked or stuck app → restart it
- **Failure action**: `kubectl` kills and recreates container
- **Example**: `/health/live` — basic liveness (process is alive)

```yaml
livenessProbe:
  httpGet: { path: /health/live, port: 8080 }
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

### Readiness Probe — "Should I receive traffic?"
- **Purpose**: Detect when app is not ready to serve → remove from load balancer
- **Failure action**: Pod removed from Service endpoints (no traffic, but **not restarted**)
- **Example**: `/health/ready` — checks DB connection, cache warmed, config loaded

```yaml
readinessProbe:
  httpGet: { path: /health/ready, port: 8080 }
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2
```

### Startup Probe — "Is the slow-starting app ready yet?"
- **Purpose**: Disable liveness/readiness probes until startup completes
- **Failure action**: Restart container

### Why Both?
- **Only liveness**: Container gets traffic before it's ready → 502/503 errors
- **Only readiness**: Stuck app never restarted → permanent outage
- **Both**: 
  - App slow → readiness fails → no traffic → still running → recovers
  - App deadlocked → liveness fails → container restarted

### Best Practices:
- Liveness checks should be **very lightweight** (no DB calls!)
- Readiness checks can be **more thorough** (DB, cache, downstream)
- Use **separate endpoints** for each
- Set appropriate `initialDelaySeconds` to avoid premature kills during warmup

---

## 22. How Would You Handle Service Discovery Failure?

### What It Means:
- Service registry (Consul, Eureka, Kubernetes DNS) is unreachable
- Stale / wrong service list
- Network partition between service and registry

### Strategies:

**1. Client-Side Caching**
```java
// Eureka — local cache survives brief registry outage
eureka:
  client:
    registry-fetch-interval-seconds: 30
    eureka-service-url-poll-interval-seconds: 300
```

**2. Multiple Registry Endpoints**
```yaml
eureka:
  client:
    service-url:
      defaultZone: "http://eureka1:8761/eureka,http://eureka2:8762/eureka,http://eureka3:8763/eureka"
```

**3. Kubernetes DNS (Built-in)**
- Service discovery via DNS is highly available
- Pod can resolve `my-service.default.svc.cluster.local` even during API server issues

**4. Fallback to Static Endpoints**
```yaml
# Resilience4j — fallback to known-good endpoint
resilience4j:
  fallback:
    instances:
      paymentService:
        fallbackMethod: "staticPaymentEndpoint"
```

**5. Health Checks + Circuit Breaker**
```java
if (!serviceRegistry.isHealthy()) {
    return cachedServiceList.get(services);   // use last-known-good
}
```

**6. Retry + Exponential Backoff**
```java
RetryConfig config = RetryConfig.custom()
    .maxAttempts(5)
    .intervalFunction(IntervalFunction.ofExponentialBackoff(1000, 2))
    .build();
```

---

## 23. What Happens When an API Gateway Goes Down?

### Impact:
- **All external traffic blocked** (gateway is single entry point)
- Internal service-to-service calls still work
- Clients see connection errors / 502 / 504

### Root Causes:
- Gateway process crashed
- Gateway saturated (CPU, memory, connections)
- Network issue between gateway and upstream
- Misconfiguration (bad route, invalid cert)

### Immediate Mitigation:
1. **Health check the gateway** — is it up?
2. **Restart / redeploy** gateway
3. **Check upstream connectivity** — can gateway reach services?
4. **Enable backup gateway** (if HA setup)

---

## 24. How Would You Design Multiple API Gateway Instances?

### Architecture:
```
Internet → Load Balancer (L4/L7) → [Gateway-1, Gateway-2, Gateway-3] → Services
```

### Key Components:

**1. L4/L7 Load Balancer in Front**
- AWS ALB / Nginx / HAProxy
- Distributes traffic across gateways
- Performs TLS termination (optional)

**2. Stateless Gateway Instances**
- No session affinity needed (use JWT or signed cookies)
- Any instance can handle any request
- Easy horizontal scaling

**3. Shared Configuration**
- Config service (Consul, Spring Cloud Config)
- All gateways pull same routes/config
- Watch for changes → reload

**4. Health Checks + Auto-Scaling**
```yaml
# Kubernetes HPA for gateway
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
minReplicas: 3
maxReplicas: 20
```

**5. Rate Limiting Distributed**
- Use Redis-backed rate limiter (not local)
```java
RateLimiter rateLimiter = RateLimiter.builder("api")
    .limitForPeriod(1000)
    .limitRefreshPeriod(Duration.ofSeconds(1))
    .distributed(true)   // Redis-backed
    .build();
```

**6. Blue/Green or Canary for Gateway Upgrades**
```yaml
# Deploy new gateway version, shift 5% traffic, monitor, shift more
```

**7. Circuit Breaker at Gateway**
- Detect unhealthy upstream → return cached response or 503
- Don't keep retrying if upstream is down

---

## 25. How Do You Handle Distributed Transactions?

### The Problem:
- Multiple services/databases need to update together
- No global 2PC across services (slow, not scalable)
- Network can fail between steps

### Solutions:

### Solution 1: Saga Pattern (Most Common)

**Choreography (event-driven):**
```
Order Service → emits OrderCreated
Payment Service → listens → charges → emits PaymentCompleted
Inventory Service → listens → reserves → emits InventoryReserved
Shipping Service → listens → schedules shipping

If Payment fails → emits PaymentFailed
Order Service listens → emits OrderCancelled
Inventory listens → releases
```

**Orchestration (central coordinator):**
```java
public class OrderSagaOrchestrator {
    
    public void execute(OrderRequest req) {
        SagaTransaction saga = sagaManager.begin();
        
        saga.addStep("reserveInventory", 
            () -> inventoryService.reserve(req.getItems()),
            () -> inventoryService.release(reservationId));
        
        saga.addStep("chargePayment",
            () -> paymentService.charge(req.getPayment()),
            () -> paymentService.refund(paymentId));
        
        saga.addStep("createOrder",
            () -> orderService.create(req),
            () -> orderService.cancel(orderId));
        
        saga.execute();
    }
}
```

### Solution 2: Two-Phase Commit (Use Sparingly)
- Coordinator + participants
- Blocking, single point of failure
- Only for cross-database within same trust boundary

### Solution 3: Outbox Pattern
```java
@Transactional
public void createOrder(OrderRequest req) {
    // 1. Save order
    orderRepository.save(order);
    
    // 2. Save outbox event (in same TX)
    outboxRepository.save(new OutboxEvent(
        "ORDER_CREATED", 
        order.getId(), 
        serialize(req))
    );
    
    // Separate process polls outbox and publishes to Kafka
}
```

### Solution 4: Eventual Consistency + Idempotency
- Accept that data may be temporarily inconsistent
- Use idempotent operations + retries
- Compensating actions for failures

---

## 26. What Happens When One Step of a Saga Fails?

### Compensation Flow:
```
Step 1: Reserve Inventory ✅
Step 2: Charge Payment ❌ (declined)
Step 3: Release Inventory (compensation for Step 1) ✅
Step 4: Notify Customer "Payment Failed"
```

### Compensation Must Be Idempotent
```java
public void releaseInventory(String reservationId) {
    // Idempotent — multiple calls safe
    reservationRepository.findById(reservationId)
        .ifPresent(r -> {
            if (!r.isReleased()) {
                r.setReleased(true);
                reservationRepository.save(r);
                inventoryService.incrementStock(r.getItems());
            }
        });
}
```

### Handling Compensation Failure:
- **Retry** compensation with backoff
- **Manual intervention** queue if retry fails
- **Alert** operations team
- **Audit log** of all compensation attempts

### Saga Timeout:
```java
SagaOptions options = SagaOptions.builder()
    .timeout(Duration.ofMinutes(5))
    .compensationTimeout(Duration.ofMinutes(2))
    .build();
```

---

## 27. Choreography vs Orchestration – Which Would You Choose?

### Choreography
- Services communicate via events
- No central coordinator
- Each service knows its own logic

**Pros:**
- Loosely coupled
- No single point of failure
- Scales well

**Cons:**
- Hard to understand overall flow (spaghetti)
- Difficult to add new steps
- Cyclic dependencies

### Orchestration
- Central saga orchestrator coordinates steps
- Calls each service in order
- Handles compensation

**Pros:**
- Easy to understand flow
- Easy to add/modify steps
- Better for complex business logic

**Cons:**
- Orchestrator is a single point of failure (mitigate with HA)
- Tight coupling to orchestrator's API

### Decision Matrix:

| Scenario | Choice |
|----------|--------|
| Simple, < 3 services | Choreography |
| Complex business logic | Orchestration |
| Need visibility into flow | Orchestration |
| Want loose coupling | Choreography |
| Many services, simple steps | Choreography |
| Banking / finance (audit) | Orchestration |

### My Choice: **Orchestration** for most business workflows (clearer, easier to maintain). **Choreography** for data propagation (CDC, event sourcing).

---

## 28. How Do You Handle Eventual Consistency?

### What It Means:
- After a write, reads may return stale data for a short period
- Common in distributed systems (replication, caching, async messaging)

### Strategies:

**1. Read-Your-Writes Consistency**
```java
// After write, force read from primary (not replica)
public Optional<Order> findById(String id, boolean isReadAfterWrite) {
    if (isReadAfterWrite) {
        return primaryDb.findById(id);   // strong consistency
    }
    return replicaDb.findById(id);        // may be stale
}
```

**2. Versioning + Conflict Detection**
```java
@Entity
public class Product {
    @Version
    private Long version;
}
// OptimisticLockException if conflict
```

**3. CRDT (Conflict-free Replicated Data Types)**
- Counters, sets, registers designed for concurrent updates
- No coordination needed

**4. Saga Compensation**
- If downstream fails, undo upstream changes

**5. Read Repair / Anti-Entropy**
- Background job reconciles replicas
- Compare hashes, fix differences

**6. User Awareness**
- Show "Updated 5 seconds ago"
- Refresh button / auto-refresh after write
- Set proper user expectations

### Banking Use Case:
- After money transfer, source account: **strong consistency** (read from primary)
- After money transfer, recipient account: **eventual consistency** OK (will reflect in seconds)

---

## 29. What Happens When Network Latency Suddenly Increases?

### Symptoms:
- All requests get slower
- Thread pools fill up (waiting for responses)
- Connection timeouts
- Cascading failures

### Root Causes:
- ISP / cloud provider issue
- DNS slowdown
- TCP retransmissions (packet loss)
- Cross-region link issue
- Congested switch / NIC

### Detection:
```yaml
# Alert on latency percentiles
alert: APILatencyP99High
expr: histogram_quantile(0.99, http_request_duration_seconds) > 1
for: 5m
```

### Mitigation:

**1. Timeouts (Fail Fast)**
- Reduce timeout temporarily to free threads

**2. Circuit Breakers (Don't Retry Forever)**
- Open circuit → don't keep waiting for slow service

**3. Load Shedding**
```java
if (activeThreads.get() > maxThreads * 0.8) {
    throw new TooManyRequestsException();   // 429
}
```

**4. Geographic Failover**
- Route traffic to different region

**5. Reduce Cross-Region Calls**
- Cache aggressively during incident
- Use local replicas

**6. Backpressure**
Backpressure propagates load signals upstream so slow consumers don't get overwhelmed:
- **Slow down producers, not consumers** — let the producer know to wait/queue, never block the consumer thread waiting on a queue that grows unbounded.
- **Bounded queues** with rejection policies (`Abort`, `CallerRuns`, `Discard`) — never use `LinkedBlockingQueue` without a size cap.
- **Semaphore-based throttling** — `Semaphore(n)` to limit concurrent in-flight requests to a downstream dependency.
- **Reactive streams** (Reactor/RxJava) — built-in backpressure via `request(n)` from consumer to producer.
- **HTTP 429 + Retry-After** — when over capacity, return `429 Too Many Requests` with `Retry-After` header so clients back off.
- **Kafka consumer `max.poll.records`** — limit per-poll batch; commits act as natural backpressure.
- **TCP flow control** — `net.ipv4.tcp_window_scaling` and `tcp_mem` for socket-level backpressure.

**Example — bounded thread pool with CallerRuns policy:**
```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    10, 10, 0L, TimeUnit.MILLISECONDS,
    new ArrayBlockingQueue<>(100),          // bounded!
    new ThreadPoolExecutor.CallerRunsPolicy() // slow producer down instead of dropping
);
```

---

## 30. How Would You Troubleshoot Intermittent API Failures?

### Intermittent = Hard to Reproduce
- Could be race condition
- Could be resource exhaustion (GC, connection pool)
- Could be external dependency flapping
- Could be network blip

### Step-by-Step Approach:

**1. Gather Data**
- Error rates per endpoint, per instance
- Correlation with system metrics (CPU, memory, GC, network)
- Time of day patterns
- Specific user/tenant patterns

**2. Look for Patterns**
```promql
# Are errors correlated with GC pauses?
increase(api_errors_total[5m]) and increase(jvm_gc_pauses_total[5m])

# Specific instance?
rate(api_errors_total[5m]) by (instance)
```

**3. Distributed Tracing**
- Use trace ID to find slow/failed requests
- Jaeger / Zipkin / Tempo

**4. Thread Dumps + Heap Dumps**
- Take multiple dumps over time
- Look for recurring threads / objects

**5. Network Analysis**
- `mtr`, `tcpdump`, packet captures
- Look for retransmits, DNS failures

**6. Load Testing**
- Try to reproduce in staging
- Use production traffic replay (e.g., Gor)

**7. Canary Analysis**
- New version vs old — is new version more flaky?
- Compare error rates between versions

### Common Root Causes:
- **Connection pool exhaustion** under peak load
- **GC pauses** > request timeout
- **Race condition** in shared state
- **DNS cache** stale entries
- **Upstream rate limit** triggered
- **Clock skew** between services (JWT expiry)

---

## 31. How Do You Trace a Request Across Multiple Services?

### Distributed Tracing Tools:
- **OpenTelemetry** (CNCF standard)
- **Jaeger** / **Zipkin** / **Tempo**
- **AWS X-Ray**, **GCP Cloud Trace**

### How It Works:
```
Client → API Gateway → Auth Service → Order Service → Payment Service → Database
   trace_id=abc       same trace_id           same trace_id
                       span_id=123              span_id=456 (child)
```

### Implementation (Spring Boot + OpenTelemetry):
```java
@Bean
public OpenTelemetry openTelemetry() {
    return AutoConfiguredOpenTelemetrySdk.builder()
        .setResultAsGlobal()
        .build();
}

@RestController
public class OrderController {
    
    @Autowired Tracer tracer;
    
    @PostMapping("/orders")
    public Order create(@RequestBody OrderRequest req) {
        Span span = tracer.spanBuilder("create-order").startSpan();
        try (Scope scope = span.makeCurrent()) {
            // propagation happens automatically via HTTP headers
            Order order = orderService.create(req);
            span.setAttribute("order.id", order.getId());
            return order;
        } finally {
            span.end();
        }
    }
}
```

### Key Concepts:
- **Trace**: End-to-end request journey
- **Span**: Single operation (one service call)
- **Context Propagation**: `traceparent` HTTP header
- **Sampling**: Don't trace 100% (too expensive)

```yaml
# Sample 10% of requests, 100% of errors
sampling:
  default:
    type: ParentBased
    root:
      type: TraceIdRatio
      argument: 0.1
    remoteParentSampled: AlwaysOn
    remoteParentNotSampled: AlwaysOff
```

### What You Get:
- Full request path visualization
- Per-service latency breakdown
- Error attribution
- Dependency map
- Bottleneck identification

---

## 32. How Do You Identify Which Service Is Causing Latency?

### Using Distributed Tracing:
1. Open Jaeger/Tempo UI
2. Filter slow traces (e.g., > 1s)
3. Look at the **Gantt chart** — long bars = slow services
4. Click span → see breakdown (DB call, external API, etc.)

### Using Service Map:
- Jaeger / Tempo shows dependencies
- Edge thickness = traffic
- Edge color/labels = error rate or latency

### Using Metrics:
```promql
# p99 latency per service
histogram_quantile(0.99,
  sum by (service, le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)

# Compare to baseline
histogram_quantile(0.99, ...) - histogram_quantile(0.99, ...) offset 1d
```

### Flame Graphs:
- Visualize where time is spent within a single request
- Wide = hot path

### APM Tools:
- **Datadog APM**, **New Relic**, **Dynatrace**
- Auto-instrumentation
- AI-based anomaly detection
- Service maps with health indicators

### Custom Profiling:
- **Continuous profiling** (Pyroscope, async-profiler for JVM)
- Always-on CPU profiling with low overhead
- See exact code paths consuming time

---

## 33. What Happens When Logs Are Missing During an Incident?

### Why Logs Go Missing:
- Disk full on log host
- Log shipper (Fluentd, Filebeat) down
- Centralized logging (ELK) overloaded
- Wrong log level configured (DEBUG off)
- Container stdout not captured
- Log rotation deleted files

### Pre-Incident: Make Logging Resilient

**1. Structured Logging**
```java
log.info("Payment processed",
    kv("orderId", orderId),
    kv("amount", amount),
    kv("customerId", customerId),
    kv("durationMs", duration));
```

**2. Centralized Logging (ELK, Loki, Splunk)**
- Don't rely on local files
- All logs ship to central store

**3. Correlation IDs**
```java
MDC.put("traceId", traceId);
MDC.put("requestId", requestId);
log.info("Processing payment");
```

**4. Multiple Log Sinks**
- Logs to stdout (k8s) + file + remote
- Redundancy

**5. Sampling Decisions**
- Trace 100% of errors
- Trace 10% of successes

### During Incident — When Logs Are Missing:

**1. Use Metrics (more reliable)**
- Prometheus, Datadog metrics still work
- Dashboards, alerts

**2. Use Traces (Distributed Tracing)**
- If tracing infra is up

**3. Use APM / RUM**
- Application performance monitoring captures requests even without logs

**4. Real-Time Commands**
- `kubectl exec` into pod → check running state
- `jstack` for live thread dump

**5. Reconstruct from Audit Tables**
- DB audit trail
- CDC events

---

## 34. How Would You Handle a Sudden Traffic Spike?

### Step 1: Identify the Cause
- Legitimate spike (viral content, news)?
- Bot attack / DDoS?
- Retry storm from upstream failure?
- Bug causing infinite loop?

### Step 2: Immediate Actions

**Rate Limiting:**
```java
@RateLimiter(name = "api", fallbackMethod = "rateLimited")
public Response handle(Request req) { ... }

public Response rateLimited(Request req, Exception e) {
    return Response.status(429).header("Retry-After", "60").build();
}
```

**Auto-Scaling:**
```yaml
# Kubernetes HPA
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```

**Load Shedding:**
- Reject low-priority traffic
- Serve cached responses for hot data
- Disable non-essential features

**DDoS Protection:**
- WAF rules (Cloudflare, AWS Shield)
- IP-based rate limiting
- CAPTCHA for suspicious traffic

### Step 3: Capacity

**Cache Aggressively:**
```java
@Cacheable(value = "hot-product", key = "#id", unless = "#result == null")
public Product getProduct(String id) { ... }
```

**Read Replicas:**
- Route reads to replicas
- Free primary for writes

**Async Processing:**
- Move non-critical work to background queues
- Don't block user requests

### Step 4: Long Term
- Capacity planning
- Pre-warm caches before known events
- Multi-region failover

---

## 35. How Do You Prevent One Tenant from Consuming All Resources?

### Multi-Tenancy Resource Isolation:

**1. Per-Tenant Rate Limiting**
```java
RateLimiter tenantLimiter = RateLimiter.builder("tenant-" + tenantId)
    .limitForPeriod(100)             // 100 req/sec per tenant
    .limitRefreshPeriod(Duration.ofSeconds(1))
    .build();
```

**2. Tenant-Aware Thread Pools (Bulkhead per Tenant)**
```java
Map<String, ThreadPoolBulkhead> tenantBulkheads = new ConcurrentHashMap<>();

ThreadPoolBulkhead getBulkhead(String tenantId) {
    return tenantBulkheads.computeIfAbsent(tenantId, id ->
        ThreadPoolBulkhead.of(id, ThreadPoolBulkheadConfig.custom()
            .coreThreadPoolSize(5)
            .maxThreadPoolSize(10)
            .queueCapacity(50)
            .build())
    );
}
```

**3. Database Quotas**
```sql
-- Per-tenant connection limit
ALTER ROLE tenant_123 CONNECTION LIMIT 10;

-- Per-tenant storage quota (PostgreSQL)
-- Use schema-per-tenant with disk quotas
```

**4. Fair Scheduling (Priority Queues)**
- High-tier tenants → higher priority
- Free tier → limited resources

**5. Cost Attribution**
- Track per-tenant: API calls, DB queries, storage
- Charge accordingly or enforce hard limits

**6. Kubernetes Namespaces**
```yaml
# Per-tenant namespace with ResourceQuota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: tenant-acme
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    pods: "50"
```

**7. Caching Per Tenant (Don't Let One Tenant's Cache Evict Others)**
- Use cache regions per tenant
- Or separate Redis instance per tier

---

## 36. How Would You Deploy a Fix Without Downtime?

### Zero-Downtime Deployment Strategies:

**1. Rolling Update (Default in Kubernetes)**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0       # never below desired count
    maxSurge: 1             # one extra pod at a time
```
- New pod starts → old pod terminates, one at a time
- Brief overlap of two versions

**2. Blue-Green Deployment**
```
Load Balancer → [Blue v1]  ← all traffic
                [Green v2]  ← deploy, test, switch
```
- Deploy new version (Green)
- Test Green independently
- Switch load balancer → Green
- Keep Blue for quick rollback

**3. Canary Deployment**
```
v1: 95% traffic
v2: 5% traffic  → monitor metrics
v1: 50% / v2: 50%
v2: 100%
```
- Roll out gradually
- Watch error rates, latency
- Roll back if regression

**4. Feature Flags**
```java
if (featureFlags.isEnabled("new-checkout-flow", userContext)) {
    return newCheckoutFlow(request);
} else {
    return oldCheckoutFlow(request);
}
```
- Deploy new code (disabled)
- Enable for % of users
- Disable if issues

**5. Database Migrations**
- **Expand-Migrate-Contract** pattern:
  1. Add new column (backward compatible)
  2. Deploy code that writes to both old + new
  3. Backfill data
  4. Deploy code that reads from new
  5. Drop old column

**6. Rolling Restart with Readiness Probes**
- Ensure new pod is ready before old pod terminates

---

## 37. How Would You Rollback a Failed Deployment?

### Automated Rollback:
```yaml
# Kubernetes — automatic rollback on failed probes
spec:
  strategy:
    type: RollingUpdate
    rollbackTo:
      revision: 0   # 0 = previous revision
```

### Manual Rollback:
```bash
# Kubernetes
kubectl rollout undo deployment/my-app
kubectl rollout undo deployment/my-app --to-revision=3

# Helm
helm rollback my-release 1

# Check status
kubectl rollout status deployment/my-app
```

### Database Rollback Considerations:
- **Forward-only migrations** — schema changes may not be reversible
- Keep migration **backward compatible** so rollback works
- Use tools like **Flyway**, **Liquibase**

### Quick Rollback Checklist:
1. Trigger rollback (kubectl/helm/CI)
2. Verify pods are healthy on old version
3. Check error rates dropped
4. Verify metrics returned to normal
5. Post-mortem on what went wrong

### Preventive Measures:
- **Health checks** before declaring deploy successful
- **Canary analysis** with auto-abort on regression
- **Database migration** tested in staging
- **Feature flags** for quick disable without rollback

---

## 38. How Do You Handle Backward Compatibility Between Services?

### API Versioning:

**1. URI Versioning**
```
/api/v1/orders
/api/v2/orders
```

**2. Header Versioning**
```
Accept: application/vnd.myapi.v2+json
```

**3. Query Parameter**
```
/api/orders?version=2
```

### Strategies for Backward Compatibility:

**1. Additive Changes (Always Safe)**
- ✅ Add new optional field to request
- ✅ Add new field to response
- ✅ Add new endpoint
- ✅ Add new enum value (clients ignore unknowns)

**2. Breaking Changes (Require Versioning)**
- ❌ Remove field
- ❌ Rename field
- ❌ Change field type
- ❌ Change semantics
- ❌ Remove endpoint

### Implementation Pattern:
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 { ... }

@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 { 
    // new logic, delegates to same service
}

// Or single controller with version mapping
@GetMapping(value = "/orders", headers = "X-API-VERSION=2")
public OrderV2 getOrderV2(...) { ... }
```

### Deprecation Process:
1. Mark old version as `@Deprecated`
2. Add `Sunset` header in response
3. Communicate timeline to clients
4. Keep both versions running during transition
5. Monitor usage of old version
6. Decommission when traffic drops to zero

### Compatibility Best Practices:
- Use **tolerant readers** (ignore unknown fields)
- Don't reuse HTTP status codes with different meanings
- Document changes in **API changelog**
- Use **OpenAPI/Swagger** for contract definition
- Run **contract tests** in CI (Pact, Spring Cloud Contract)
- Prefer **event-driven** for inter-service (decouples versions)

---

## 39. How Would You Design a Microservice to Survive Dependency Failures?

### Design Principles:

**1. Defense in Depth**
```
┌─────────────────────────────────┐
│  Timeout (fail fast)            │
│   ┌─────────────────────────┐   │
│   │  Circuit Breaker        │   │
│   │   ┌─────────────────┐   │   │
│   │   │  Bulkhead       │   │   │
│   │   │   ┌─────────┐   │   │   │
│   │   │   │ Retry   │   │   │   │
│   │   │   │  ┌───┐  │   │   │   │
│   │   │   │  │API│  │   │   │   │
│   │   │   │  └───┘  │   │   │   │
│   │   │   └─────────┘   │   │   │
│   │   └─────────────────┘   │   │
│   └─────────────────────────┘   │
└─────────────────────────────────┘
```

**2. Async Where Possible**
```java
@Async
public CompletableFuture<Recommendation> getRecommendations(String userId) {
    // If this fails, user still gets base page
    return CompletableFuture.supplyAsync(() -> 
        recommendationService.compute(userId));
}
```

**3. Caching**
- Cache critical responses (Redis, in-memory)
- TTL-based invalidation
- Don't fail if cache is down (degrade to source)

**4. Fallback Strategies**
- **Stale data** → return last-known value
- **Default value** → safe placeholder
- **Cached page** → for read-heavy paths
- **Queue for later** → for write paths (outbox)

**5. Graceful Degradation Pattern**
```java
public ProductDetail getProduct(String id) {
    Product product = productService.getById(id);
    ProductDetail detail = new ProductDetail(product);
    
    try {
        detail.setReviews(reviewService.getReviews(id));     // optional
    } catch (Exception e) {
        detail.setReviews(List.of());  // skip reviews
    }
    
    try {
        detail.setRecommendations(recService.get(id));        // optional
    } catch (Exception e) {
        // skip
    }
    
    return detail;   // always return SOMETHING
}
```

**6. Self-Preservation**
- Shed load before consuming all resources (load shedding)
- Reject requests when own health degrades
- Don't accept work you can't complete

**7. Bulkheading (Per-Dependency Pools)**
- Separate thread pool per downstream
- One slow dep doesn't starve others

**8. Observability**
- Know when a dependency is degraded
- Alert on circuit-open events
- Track fallback rate

---

## 40. A Production Order Flow Is Failing Randomly. How Would You Investigate It End-to-End?

### Investigation Framework: **Observe → Orient → Decide → Act**

### Phase 1: Define the Problem (5 min)
- What's "failing"? (5xx, wrong data, timeout?)
- For which users / when / how often?
- Started when? (Correlate with deploys, config changes)
- One region / instance / customer or global?

### Phase 2: Gather Signals (15 min)

**1. Logs**
```bash
# Filter errors for the affected flow
kubectl logs -l app=order-service --since=30m | grep -i error

# Centralized (Kibana)
# Filter: service:order-service AND level:ERROR AND traceId:*
```

**2. Metrics**
```promql
# Error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) by (endpoint)

# Latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Per-instance (is one pod bad?)
sum by (instance)(rate(http_requests_total{status=~"5.."}[5m]))
```

**3. Distributed Traces**
- Find slow / failed traces in Jaeger
- Identify the slow / erroring span (which service?)
- Drill into that service

**4. Recent Changes**
```bash
git log --since="2 days ago" --oneline
kubectl rollout history deployment/order-service
```

### Phase 3: Drill Down (20 min)

| Finding | Next Step |
|---------|-----------|
| All errors from one service | Check that service's logs/metrics |
| One instance bad | Check pod health, host issues |
| Database slow | Check `pg_stat_activity`, slow query log |
| Specific customer | Check tenant config, limits |
| Started after deploy | Check what changed |

**Common Root Causes:**
- **Timeout too low** (p99 latency > timeout)
- **Connection pool exhausted** (DB / downstream)
- **GC pause** (large heap, memory leak)
- **Downstream degradation** (slow upstream)
- **Race condition** (intermittent by nature)
- **Resource limit hit** (CPU throttling, OOM)

### Phase 4: Mitigation (Immediate)
1. **Rollback** if deploy-related
2. **Scale up** if capacity issue
3. **Increase timeout** if misconfigured
4. **Restart bad pod** if single-instance
5. **Disable feature flag** if feature-specific
6. **Open circuit breaker** if downstream bad

### Phase 5: Long-Term Fix
- Add regression test
- Add monitoring/alerting
- Improve error handling
- Capacity planning
- Document in runbook

### Communication:
- Update incident channel every 15 min
- Notify customer support
- Status page if user-facing
- Post-mortem after resolution

### Key Takeaway
> Random failures are usually **deterministic** — they're triggered by specific conditions (peak load, specific input, GC). Use **observability** to find the trigger, then fix the root cause. Don't just restart and hope.
