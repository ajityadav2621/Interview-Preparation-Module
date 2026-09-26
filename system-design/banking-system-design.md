# System Design Interview Questions — Banking & High-Throughput Systems

## Q20. Design a High-Throughput Transaction Processing System with Strong Consistency Guarantees

### Requirements

**Functional:**
- Process 100,000+ transactions/sec
- Account balance updates (debit/credit/transfer)
- Transaction history
- Account statements

**Non-Functional:**
- **Strong consistency** — balance must never be wrong
- **Zero data loss** — regulatory requirement
- **Low latency** — p99 < 100ms
- **High availability** — 99.999%
- **Auditability** — every change logged

### High-Level Architecture

```
                     ┌─────────────────┐
                     │  Load Balancer  │
                     └────────┬────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
         [API GW 1]      [API GW 2]      [API GW 3]
              │               │               │
              └───────────────┼───────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
       [Txn Service]   [Txn Service]   [Txn Service]    ← Stateless
              │               │               │
              └───────────────┼───────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
       [Primary DB]   [Sync Replica]   [Async Replica]
         (Writes)        (Reads)         (Analytics)
              │
       [Ledger DB]   ← Append-only, audit trail
              │
       [Event Bus]   ← Kafka for downstream
```

### Core Components

**1. API Layer (Stateless)**
```java
@RestController
@RequestMapping("/api/v1/transactions")
public class TransactionController {
    
    @PostMapping("/transfer")
    @Idempotent  // Custom annotation — backed by IdempotencyService (see Q12 in spring-boot-interview-questions.md)
    public TransferResponse transfer(
            @RequestHeader("Idempotency-Key") String idempKey,
            @RequestBody @Valid TransferRequest request) {
        
        return transactionService.processTransfer(idempKey, request);
    }
}

/**
 * Custom annotation — wired by an AOP aspect or a HandlerInterceptor that
 * delegates to IdempotencyService.execute(...). Definition (sketch):
 *
 * @Target(METHOD) @Retention(RUNTIME)
 * public @interface Idempotent {
 *     String header() default "Idempotency-Key";
 *     long ttlSeconds() default 86400;
 * }
 */
```

**2. Transaction Service (Saga Orchestrator)**
```java
@Service
public class TransactionService {
    
    @Transactional
    public TransferResponse processTransfer(String idempKey, TransferRequest req) {
        // 1. Reserve funds atomically
        accountService.debitAtomic(req.fromAccount, req.amount);
        
        try {
            // 2. Credit destination
            accountService.creditAtomic(req.toAccount, req.amount);
            
            // 3. Append to ledger
            ledgerService.append(req);
            
            // 4. Save outbox event
            outboxService.save(new TransactionCompletedEvent(req));
            
            return TransferResponse.success(req.transactionId);
        } catch (Exception e) {
            // Compensate
            accountService.creditAtomic(req.fromAccount, req.amount);
            throw e;
        }
    }
}
```

**3. Database (Sharded PostgreSQL)**
```
Sharding strategy: hash(account_id) % N shards
- 16 shards initially, scale to 64
- Each shard = primary + 2 replicas
- Cross-shard transactions via 2PC (rare)
```

**4. Cache Layer (Read-Through)**
```java
@Cacheable(value = "balance", key = "#accountId")
public BigDecimal getBalance(String accountId) {
    return accountRepository.getBalance(accountId);
}
```

**5. Event Streaming (Kafka)**
```java
// Publish completed transactions
kafkaTemplate.send("transactions", transactionKey, txnCompletedEvent);
```

### Consistency Mechanisms

**Atomic Balance Updates:**
```sql
-- Compare-and-swap with version (optimistic concurrency control)
-- :expectedVersion is the version previously read by the caller (e.g., in a SELECT
-- at the start of the request). If another transaction modified the row in
-- between, the WHERE clause fails and rowsAffected = 0.
-- The CAS check still catches concurrent modifications between the SELECT
-- and this UPDATE, even if they occurred milliseconds apart.
UPDATE accounts
SET balance = balance - :amount,
    version = version + 1
WHERE id = :accountId
  AND balance >= :amount
  AND version = :expectedVersion
RETURNING balance, version;
```

**Distributed Lock for Cross-Account Transfers:**
```java
@Transactional
public void transfer(String from, String to, BigDecimal amount) {
    // Always lock in same order — avoid deadlock
    List<String> sortedAccounts = Stream.of(from, to).sorted().toList();
    
    Account first = lockAndLoad(sortedAccounts.get(0));
    Account second = lockAndLoad(sortedAccounts.get(1));
    
    // Validate, debit, credit, ledger entry — atomic
}
```

**Idempotency:**
```sql
CREATE TABLE processed_requests (
    idempotency_key VARCHAR(64) PRIMARY KEY,
    request_hash CHAR(64) NOT NULL,
    response JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Scalability Features

**1. Partitioning (PostgreSQL declarative):**
```sql
CREATE TABLE ledger (
    id BIGSERIAL,
    account_id BIGINT NOT NULL,
    txn_id BIGINT NOT NULL,
    amount DECIMAL(19, 4) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Monthly partitions
CREATE TABLE ledger_2024_01 PARTITION OF ledger
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

**2. Async Outbox Pattern:**
```java
@Transactional
public void onTransactionComplete(Transaction txn) {
    ledgerRepository.save(txn);                  // ACID
    outboxRepository.save(toOutboxEvent(txn));    // Same TX
}

// Separate worker polls outbox and publishes to Kafka
@Scheduled(fixedDelay = 500)
public void publishOutbox() {
    outboxRepository.findUnpublished()
        .forEach(event -> {
            kafkaTemplate.send(event.getTopic(), event.getPayload());
            outboxRepository.markPublished(event.getId());
        });
}
```

**3. Read Replicas with Stale-Read Tolerance:**
- Account balance reads → primary (strong consistency)
- Transaction history reads → replica (eventual OK)

### Failure Handling

| Scenario | Strategy |
|----------|----------|
| DB primary down | Failover to replica (Patroni, etcd) |
| Slow downstream | Circuit breaker + timeout |
| Network partition | Quorum-based decision (avoid split brain) |
| Data corruption | Restore from PITR backup |
| Kafka unavailable | Outbox buffers events |

### Performance Optimizations

- **Connection pooling** (HikariCP)
- **Prepared statements** (cached query plans)
- **Batch inserts** for ledger
- **Asynchronous logging**
- **Off-heap caching** (Redis for hot balances)
- **Read replica routing** for non-critical reads

### Key Takeaways
- **Atomic SQL updates** = strongest single-row guarantee
- **Saga + 2PC** for cross-shard transactions
- **Idempotency keys** for safe retries
- **Append-only ledger** for audit + time-travel
- **Sync replication** for critical, async for analytics

---

## Q21. Design a Distributed Caching Strategy for Frequently Accessed Account Balance Data

### Requirements
- p99 read latency < 10ms for balance lookups
- 10M+ balance lookups/sec
- Strong consistency on balance updates
- Survive cache failures

### Architecture

```
┌──────────────────────────────────────────────────┐
│              Account Service                     │
├──────────────────────────────────────────────────┤
│  1. Read-through from Redis                      │
│  2. On miss: read from DB → populate cache       │
│  3. On write: invalidate cache + DB update       │
└──────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────┐
│        Redis Cluster (Sharded)                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│  │ Master 1 │ │ Master 2 │ │ Master 3 │         │
│  │ + Slave  │ │ + Slave  │ │ + Slave  │         │
│  └──────────┘ └──────────┘ └──────────┘         │
└──────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────┐
│            PostgreSQL Primary                    │
│       (Source of truth, durable)                 │
└──────────────────────────────────────────────────┘
```

### Multi-Level Cache

**L1: In-Process Cache (Caffeine)**
- Fastest: nanoseconds
- Limited size (10MB–1GB)
- Short TTL (1-5 seconds)

**L2: Distributed Cache (Redis)**
- Fast: ~1ms
- Large size (GB-TB)
- Longer TTL (minutes)

**L3: Database**
- Slowest: ~10ms
- Unlimited size
- Source of truth

### Implementation

```java
@Service
public class AccountService {
    
    private final Cache<String, BigDecimal> localCache;     // L1
    private final RedisTemplate<String, BigDecimal> redis;  // L2
    private final AccountRepository accountRepository;      // L3
    
    public BigDecimal getBalance(String accountId) {
        // L1: Local cache
        BigDecimal cached = localCache.getIfPresent(accountId);
        if (cached != null) return cached;
        
        // L2: Redis
        BigDecimal fromRedis = redis.opsForValue().get("balance:" + accountId);
        if (fromRedis != null) {
            localCache.put(accountId, fromRedis);
            return fromRedis;
        }
        
        // L3: Database
        BigDecimal fromDb = accountRepository.getBalance(accountId);
        if (fromDb != null) {
            redis.opsForValue().set("balance:" + accountId, fromDb, Duration.ofMinutes(5));
            localCache.put(accountId, fromDb);
        }
        return fromDb;
    }
    
    @Transactional
    public void updateBalance(String accountId, BigDecimal newBalance) {
        // 1. Update DB (source of truth)
        accountRepository.updateBalance(accountId, newBalance);
        
        // 2. Invalidate cache (write-through invalidation)
        redis.delete("balance:" + accountId);
        localCache.invalidate(accountId);
        
        // Or write-through: redis.set(... newBalance ...)
    }
}
```

### Cache Patterns

**Pattern 1: Cache-Aside (Lazy Loading)**
```java
public V get(K key) {
    V value = cache.get(key);
    if (value == null) {
        value = db.load(key);
        cache.put(key, value);
    }
    return value;
}
```
✅ Simple, only loads what's needed  
❌ First read is slow, can have stale data

**Pattern 2: Write-Through**
```java
public void put(K key, V value) {
    db.save(key, value);
    cache.put(key, value);  // synchronous
}
```
✅ Always consistent  
❌ Slower writes

**Pattern 3: Write-Behind (Write-Back)**
```java
public void put(K key, V value) {
    cache.put(key, value);            // immediate
    asyncQueue.add(() -> db.save(key, value));   // async
}
```
✅ Fast writes  
❌ Risk of data loss if cache fails

**Pattern 4: Write-Around**
```java
public void put(K key, V value) {
    db.save(key, value);
    cache.invalidate(key);  // next read will reload
}
```
✅ Good for write-heavy, read-rarely  
❌ First read after write is slow

### Choosing the Right Pattern for Balance

For **balance updates** — use **Write-Through**:
```java
@Transactional
public void updateBalance(String accountId, BigDecimal newBalance) {
    // 1. Atomic DB update
    int rows = jdbcTemplate.update(
        "UPDATE accounts SET balance = ?, version = version + 1 WHERE id = ?",
        newBalance, accountId
    );
    if (rows == 0) throw new OptimisticLockException();
    
    // 2. Update cache (write-through)
    redis.opsForValue().set("balance:" + accountId, newBalance);
    localCache.put(accountId, newBalance);
}
```

### Cache Stampede Prevention

When a hot key expires, all requests hit DB simultaneously.

**Solutions:**

**1. Lock-Based Loading (Single Flight)**
```java
private final ConcurrentHashMap<String, CompletableFuture<BigDecimal>> loaders = new ConcurrentHashMap<>();

public BigDecimal getBalance(String accountId) {
    BigDecimal cached = redis.opsForValue().get("balance:" + accountId);
    if (cached != null) return cached;
    
    // Only one thread loads; others wait
    return loaders.computeIfAbsent(accountId, id -> 
        CompletableFuture.supplyAsync(() -> {
            BigDecimal fromDb = accountRepository.getBalance(id);
            redis.opsForValue().set("balance:" + id, fromDb, Duration.ofMinutes(5));
            loaders.remove(id);
            return fromDb;
        })
    ).join();
}
```

**2. Probabilistic Early Expiration**
```java
public BigDecimal getBalance(String accountId) {
    CachedBalance entry = redis.opsForValue().get("balance:" + accountId);
    if (entry != null) {
        // Refresh early with probability based on remaining TTL
        double deltaSeconds = ChronoUnit.SECONDS.between(Instant.now(), entry.expiry);
        double deltaMs = deltaSeconds * 1000;
        double computedDelta = deltaMs * Math.log(Math.random()) / BETA;
        if (computedDelta < 0) {
            // Refresh in background
            asyncRefresh(accountId);
        }
        return entry.value;
    }
    // ... load
}
```

**3. Background Refresh**
- Worker refreshes hot keys before expiry
- Always warm cache

### Cache Invalidation Strategies

**1. TTL-Based**
```java
redis.opsForValue().set(key, value, Duration.ofMinutes(5));
```
✅ Simple, predictable  
❌ Eventually stale

**2. Event-Based Invalidation**
```java
// Publish on update
kafkaTemplate.send("cache-invalidation", accountId);

// All instances listen
@KafkaListener(topics = "cache-invalidation")
public void invalidate(String accountId) {
    redis.delete("balance:" + accountId);
    localCache.invalidate(accountId);
}
```
✅ Always consistent  
❌ More infrastructure

**3. Version-Based**
```java
public class CachedBalance {
    BigDecimal value;
    long version;
    Instant createdAt;
}

// Compare versions before returning
```

### Cache Sizing

| Cache | Size | Items |
|-------|------|-------|
| L1 (Caffeine) | 256MB | 50K-100K hot balances |
| L2 (Redis) | 16GB | 10M-50M balances |

**Hot key detection:**
- Track access frequency
- Promote frequently accessed keys to L1
- Demote rarely accessed to L2 only

### Redis Cluster Design

- **Hash Slot** = `CRC16(accountId) % 16384`
- **3 Masters + 3 Replicas** (HA)
- **Sharding by account_id** — keeps related data together
- **Consistent hashing** for rebalancing

### Key Takeaways

- **Multi-level cache** (in-process + distributed) for best performance
- **Write-through** for critical data (balances)
- **Single-flight** to prevent cache stampede
- **Versioning** for consistency
- **Monitor hit ratio** (target > 95%)
- **Plan for cache failure** — graceful degradation to DB

---

## Q22. How Would You Design an Event-Driven Audit Logging System for Financial Transactions?

### Requirements

- **Immutable** — append-only, no updates/deletes
- **Complete** — every transaction logged
- **Queryable** — find specific events
- **Compliance** — meet regulatory requirements (SOX, GDPR)
- **Real-time** — events available within seconds
- **Durable** — survive any single failure

### Architecture

```
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌─────────────┐
│  Account   │    │  Payment   │    │   Order    │    │   Loan      │
│  Service   │    │  Service   │    │  Service   │    │  Service    │
└─────┬──────┘    └─────┬──────┘    └─────┬──────┘    └──────┬──────┘
      │                 │                 │                  │
      └────────┬────────┴────────┬────────┴──────────┬───────┘
               │                 │                  │
               ▼                 ▼                  ▼
         ┌─────────────────────────────────────────────┐
         │      Event Bus (Kafka)                      │
         │  Topics:                                    │
         │  - transactions.events                      │
         │  - accounts.events                          │
         │  - audit.security.events                    │
         └─────────────────┬───────────────────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
      ┌─────────────┐ ┌─────────┐ ┌──────────────┐
      │ Audit DB    │ │ Search  │ │ Compliance   │
      │ (Append-only)│ │ Index   │ │ Dashboard    │
      │             │ │(ES)     │ │              │
      └─────────────┘ └─────────┘ └──────────────┘
```

### Event Schema

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public record AuditEvent(
    String eventId,                    // ULID — unique, sortable
    String eventType,                  // TRANSACTION_CREATED, ACCOUNT_UPDATED, etc.
    String aggregateId,                // entity affected
    String aggregateType,              // ACCOUNT, TRANSACTION, USER
    String actorId,                    // who did it
    String actorType,                  // USER, SYSTEM, ADMIN
    String sourceService,
    Instant occurredAt,                // event time (UTC)
    Instant recordedAt,                // when we captured it
    String correlationId,              // trace ID
    Map<String, Object> before,        // previous state
    Map<String, Object> after,         // new state
    Map<String, Object> metadata,      // IP, user agent, etc.
    String schemaVersion               // for evolution
) {}
```

### Event Capture (Outbox Pattern)

```java
@Entity
public class Transaction {
    @Id Long id;
    BigDecimal amount;
    // ...
    
    @OneToMany(cascade = ALL, orphanRemoval = true)
    List<DomainEvent> events = new ArrayList<>();
    
    public void execute(BigDecimal amount) {
        this.amount = this.amount.add(amount);
        events.add(new MoneyCreditedEvent(id, amount, this.balance));
    }
}

@Transactional
public void commitTransaction(Transaction txn) {
    // 1. Save transaction + outbox in SAME DB transaction
    transactionRepository.save(txn);
    outboxRepository.saveAll(txn.getEvents());
    // Both committed atomically
}

// Separate worker publishes outbox → Kafka
```

### Audit Database Design

```sql
-- Append-only audit log
CREATE TABLE audit_log (
    event_id            CHAR(26) PRIMARY KEY,    -- ULID
    event_type          VARCHAR(100) NOT NULL,
    aggregate_id        VARCHAR(50) NOT NULL,
    aggregate_type      VARCHAR(50) NOT NULL,
    actor_id            VARCHAR(50),
    actor_type          VARCHAR(20),
    source_service      VARCHAR(50) NOT NULL,
    occurred_at         TIMESTAMP(6) NOT NULL,
    recorded_at         TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    correlation_id      VARCHAR(50),
    schema_version      VARCHAR(20) NOT NULL,
    before_state        JSONB,
    after_state         JSONB,
    metadata            JSONB,
    
    -- Prevent any modification
    CONSTRAINT no_update CHECK (true)
);

-- Indexes for common queries
CREATE INDEX idx_aggregate ON audit_log(aggregate_type, aggregate_id, occurred_at DESC);
CREATE INDEX idx_actor ON audit_log(actor_id, occurred_at DESC);
CREATE INDEX idx_event_type ON audit_log(event_type, occurred_at DESC);
CREATE INDEX idx_correlation ON audit_log(correlation_id);

-- Partition by month
CREATE TABLE audit_log_2024_01 PARTITION OF audit_log
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Restrict UPDATE/DELETE via permissions
REVOKE UPDATE, DELETE ON audit_log FROM app_user;
```

### Audit Consumer → Database

```java
@Service
public class AuditConsumer {
    
    @KafkaListener(topics = "transactions.events", groupId = "audit-writer")
    public void consume(ConsumerRecord<String, AuditEvent> record) {
        AuditEvent event = record.value();
        try {
            // 1. Write to audit DB (insert-only)
            auditRepository.insert(event);
            
            // 2. Index for search
            searchService.index(event);
            
            // 3. Check compliance rules
            complianceService.check(event);
        } catch (Exception e) {
            // Send to DLT for manual review
            dltProducer.send(record);
            throw e;   // don't ack — will retry
        }
    }
}
```

### Search & Query (Elasticsearch)

```json
{
  "mappings": {
    "properties": {
      "eventId":        { "type": "keyword" },
      "eventType":      { "type": "keyword" },
      "aggregateId":    { "type": "keyword" },
      "actorId":        { "type": "keyword" },
      "occurredAt":     { "type": "date" },
      "before":         { "type": "object", "enabled": false },
      "after":          { "type": "object", "enabled": false },
      "metadata":       { "type": "object", "enabled": true }
    }
  }
}
```

### Compliance & Reporting

```java
@Service
public class ComplianceService {
    
    // Real-time rule evaluation
    @KafkaListener(topics = "transactions.events")
    public void checkRules(AuditEvent event) {
        if (event.getEventType().equals("LARGE_TRANSFER") 
            && exceedsThreshold(event.getAfter())) {
            alertService.notify("Suspicious transfer: " + event.getEventId());
            regulatoryService.reportSAR(event);
        }
    }
    
    // Periodic reports
    @Scheduled(cron = "0 0 1 * * ?")   // Daily at 1 AM
    public void generateDailyReport() {
        reportService.generate(
            "DAILY_TRANSACTION_SUMMARY",
            LocalDate.now().minusDays(1)
        );
    }
}
```

### Immutability Enforcement

**1. Database Level**
- No UPDATE/DELETE permissions on audit table
- Triggers to reject modifications

**2. Application Level**
- Audit log table has no UPDATE methods in repositories
- Code review checklist

**3. Cryptographic Verification**
```java
// Each event includes hash of previous event → tamper-evident chain
public AuditEvent {
    String eventId;
    String previousHash;    // hash of prior event
    String hash;            // hash of this event's content + previousHash
}

// Verification worker
@Scheduled(fixedDelay = 60000)
public void verifyChain() {
    String prevHash = GENESIS_HASH;
    for (AuditEvent e : auditRepository.findAll()) {
        String expectedHash = sha256(e.getContent() + prevHash);
        if (!e.getHash().equals(expectedHash)) {
            alertService.notify("TAMPERING DETECTED: " + e.getEventId());
        }
        prevHash = e.getHash();
    }
}
```

### GDPR & Data Privacy

```java
// Encryption at field level for PII
public class AuditEvent {
    @EncryptedField
    String customerName;
    
    @EncryptedField  
    String email;
}

// "Right to be forgotten" — pseudonymize, don't delete
public void anonymizeCustomer(String customerId) {
    auditRepository.anonymize(customerId, "REDACTED-" + UUID.randomUUID());
    // Audit trail remains for compliance, but PII is removed
}
```

### Key Takeaways

- **Outbox pattern** for atomic DB + event publish
- **Append-only schema** with no UPDATE/DELETE
- **Multiple consumers** (audit DB, search, compliance)
- **Cryptographic chaining** for tamper detection
- **Partitioning by date** for retention management
- **Encryption** for PII at rest

---

## Q23. Design Payment APIs That Safely Handle Retries and Duplicate Requests

### Requirements

- Idempotent — same request → same result
- Atomic — no partial state
- Auditable — every attempt logged
- Compliant — meets PCI-DSS
- Resilient — survives retries, crashes, duplicates

### Key Endpoints

```
POST /api/v1/payments        — Create payment (idempotent)
GET  /api/v1/payments/{id}   — Retrieve payment status
POST /api/v1/payments/{id}/capture  — Capture authorized payment
POST /api/v1/payments/{id}/refund   — Refund payment
```

### Request Flow

```
Client → API Gateway → Payment Service → [Idempotency Check]
                                              ↓
                                        [Balance Check]
                                              ↓
                                        [Fraud Check]
                                              ↓
                                        [Reserve Funds]
                                              ↓
                                        [External Gateway Call]
                                              ↓
                                        [Persist + Audit]
                                              ↓
                                        Return result (cache)
```

### Idempotency Implementation

**Header:**
```
POST /api/v1/payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{ "amount": 1000, "currency": "USD", "source": "card_xxx" }
```

**Server-Side:**
```java
@RestController
@RequestMapping("/api/v1/payments")
public class PaymentController {
    
    @PostMapping
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestHeader(value = "Idempotency-Key") String idempKey,
            @RequestHeader(value = "X-Request-ID") String requestId,
            @RequestBody @Valid PaymentRequest request) {
        
        // Idempotency window: 24 hours
        IdempotencyResult result = idempotencyService.executeIdempotent(
            idempKey, request, requestId, Duration.ofHours(24),
            () -> paymentProcessor.process(request)
        );
        
        return ResponseEntity.status(result.getStatus())
            .header("X-Idempotent-Replay", String.valueOf(result.isReplay()))
            .body(result.getResponse());
    }
}

@Service
public class IdempotencyService {
    
    public <T> IdempotencyResult<T> executeIdempotent(
            String key, Object request, String requestId,
            Duration ttl, Supplier<ResponseEntity<T>> action) {
        
        String fingerprint = sha256(serialize(request));
        String cacheKey = "idem:" + key;
        
        // 1. Check existing response
        StoredResponse stored = idempRepository.findByKey(key);
        
        if (stored != null) {
            // Verify request fingerprint matches
            if (!stored.getFingerprint().equals(fingerprint)) {
                throw new IdempotencyKeyConflictException(
                    "Same key used with different request body");
            }
            
            // Already completed → return cached response
            if (stored.isCompleted()) {
                return IdempotencyResult.replay(stored);
            }
            
            // In-flight → return 409 (or wait for completion)
            throw new RequestInProgressException(key);
        }
        
        // 2. Reserve the key
        IdempotencyRecord record = new IdempotencyRecord(key, fingerprint, requestId, "PENDING");
        try {
            idempRepository.save(record);
        } catch (DuplicateKeyException e) {
            // Race condition — re-fetch
            return executeIdempotent(key, request, requestId, ttl, action);
        }
        
        // 3. Execute the action
        try {
            ResponseEntity<T> response = action.get();
            
            // 4. Store result
            idempRepository.markCompleted(key, response.getBody(), ttl);
            return IdempotencyResult.fresh(response);
        } catch (Exception e) {
            // 5. Allow retry on failure — remove reservation
            idempRepository.deleteByKey(key);
            throw e;
        }
    }
}
```

### Payment State Machine

```
                    ┌──────────────┐
                    │   CREATED    │
                    └──────┬───────┘
                           │ authorize
                           ▼
                    ┌──────────────┐
                    │ AUTHORIZED   │
                    └──────┬───────┘
            capture       │         │ void
        ┌──────────────────┤         ├──────────────────┐
        ▼                  ▼         ▼                  ▼
  ┌──────────┐      ┌──────────┐  ┌────────┐    ┌──────────┐
  │ CAPTURED │      │ CAPTURED │  │ VOIDED │    │ EXPIRED  │
  └────┬─────┘      └────┬─────┘  └────────┘    └──────────┘
       │ refund           │ partial_refund
       ▼                  ▼
  ┌──────────┐      ┌──────────────┐
  │ REFUNDED │      │ PARTIALLY_   │
  └──────────┘      │ REFUNDED     │
                   └──────────────┘
```

### Atomic Money Movement

```java
@Service
public class PaymentProcessor {
    
    @Transactional
    public PaymentResponse process(PaymentRequest request) {
        // 1. Validate
        validate(request);
        
        // 2. Reserve funds atomically
        Account account = accountRepo.findByIdForUpdate(request.accountId);
        if (account.getBalance().compareTo(request.amount) < 0) {
            throw new InsufficientFundsException();
        }
        
        // 3. Create payment record
        Payment payment = new Payment(
            request.accountId,
            request.amount,
            request.currency,
            "AUTHORIZED"
        );
        payment = paymentRepository.save(payment);
        
        // 4. Call external gateway (with idempotency)
        try {
            GatewayResponse gatewayResponse = paymentGateway.authorize(
                payment.getId(),     // Use our ID as external idempotency key
                request.amount,
                request.source
            );
            
            // 5. Update with gateway response
            payment.setGatewayId(gatewayResponse.transactionId);
            payment.setStatus("AUTHORIZED");
            paymentRepository.save(payment);
            
            // 6. Reserve funds (debit hold)
            account.reserve(request.amount);
            accountRepository.save(account);
            
            // 7. Audit log
            auditService.log(AuditEvent.paymentAuthorized(payment));
            
            return PaymentResponse.success(payment);
            
        } catch (GatewayException e) {
            payment.setStatus("FAILED");
            payment.setFailureReason(e.getMessage());
            paymentRepository.save(payment);
            
            auditService.log(AuditEvent.paymentFailed(payment));
            throw new PaymentFailedException(e.getMessage());
        }
    }
}
```

### External Gateway Integration

```java
@Service
public class StripePaymentGateway {
    
    public GatewayResponse authorize(String idempKey, BigDecimal amount, PaymentSource source) {
        // Stripe has built-in idempotency — pass our key
        RequestOptions options = RequestOptions.builder()
            .setIdempotencyKey(idempKey)
            .build();
        
        return stripeClient.paymentIntents().create(params, options);
    }
}
```

### Webhook Handling (Async Confirmations)

```java
@RestController
@RequestMapping("/webhooks")
public class PaymentWebhookController {
    
    @PostMapping("/stripe")
    public ResponseEntity<String> handleStripeWebhook(
            @RequestBody String payload,
            @RequestHeader("Stripe-Signature") String signature) {
        
        // 1. Verify signature (security)
        if (!stripeWebhookVerifier.verifySignature(payload, signature)) {
            return ResponseEntity.status(401).body("Invalid signature");
        }
        
        // 2. Parse event
        Event event = Event.deserialize(payload);
        
        // 3. Idempotent processing
        String webhookId = event.getId();
        if (webhookRepository.existsById(webhookId)) {
            return ResponseEntity.ok("Already processed");
        }
        
        // 4. Process event
        try {
            switch (event.getType()) {
                case "payment_intent.succeeded":
                    paymentService.markCaptured(event);
                    break;
                case "payment_intent.payment_failed":
                    paymentService.markFailed(event);
                    break;
                // ...
            }
            webhookRepository.save(new ProcessedWebhook(webhookId, Instant.now()));
        } catch (Exception e) {
            // Don't ack — Stripe will retry
            throw e;
        }
        
        return ResponseEntity.ok("OK");
    }
}
```

### Retry Strategy with Backoff

```java
@Retryable(
    value = {TransientException.class},
    maxAttempts = 5,
    backoff = @Backoff(
        delay = 1000,
        multiplier = 2,
        maxDelay = 30000,
        randomizer = 0.5   // jitter
    )
)
public GatewayResponse callExternal(String key, BigDecimal amount) {
    return paymentGateway.charge(key, amount);
}
```

### Idempotency vs Retry

| Concern | Idempotency | Retry |
|---------|-------------|-------|
| Purpose | Prevent duplicate side-effects | Recover from transient failures |
| Where | API layer | Internal calls |
| Scope | Client-supplied key | Auto-generated |
| Lifetime | Hours/days | Seconds/minutes |

### Key Takeaways

- **Idempotency-Key header** for client retries
- **Fingerprint check** — same key, different body = error
- **External gateway** gets our internal ID as their idempotency key
- **Webhook deduplication** via event ID
- **State machine** for payment lifecycle
- **Append-only audit log** for compliance
- **Two-phase: authorize → capture** for safety
