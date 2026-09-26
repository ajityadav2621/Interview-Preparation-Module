# Database & Indexing Interview Questions

## Q16. How Do Database Indexes Work Internally and How Do You Decide Index Ordering?

### What Is an Index?

An index is a **separate data structure** that improves the speed of data retrieval at the cost of additional storage and write overhead.

Without an index → **Full Table Scan** (O(n))
With an index → **B-Tree lookup** (O(log n))

### B-Tree Index Internals

```
                    [50, 100]
                   /    |    \
              [25,40] [75,90] [125,150]
              /  |  \    |       /   \
            ... leaf nodes ... (point to actual rows)
```

**Properties:**
- **Balanced** — all leaves at same depth
- **Sorted** — keys stored in order
- **Multi-level** — root → internal → leaf
- **Page-based** — fits in disk pages (typically 16KB)

**Lookup Process:**
1. Start at root page
2. Binary search to find child page
3. Repeat until leaf page
4. Get row pointer (heap tuple ID)

### How MySQL InnoDB Index Differs

InnoDB uses **clustered index**:
- **Primary key = table itself** — rows stored in PK order
- **Secondary indexes** — store PK + indexed columns, then look up via PK

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,           -- clustered (rows in id order)
    email VARCHAR(100),
    name VARCHAR(100)
);

CREATE INDEX idx_email ON users(email);  -- secondary, stores (email, id)
```

When you query `SELECT * FROM users WHERE email = ?`:
1. Look up `email` in `idx_email` → get `id`
2. Look up `id` in clustered index → get full row
3. This is called a **bookmark lookup** or **back to table lookup**

### Index Ordering — How to Decide

**Rule 1: Most Selective Column First**
```sql
-- status has only 3 values (low selectivity) — bad first
-- customer_id has millions of unique values — good first

-- Bad
CREATE INDEX idx_bad ON orders(status, customer_id);

-- Good
CREATE INDEX idx_good ON orders(customer_id, status);
```

**Rule 2: Match Query WHERE + ORDER BY**
```sql
-- Query: WHERE customer_id = ? ORDER BY created_at DESC
CREATE INDEX idx ON orders(customer_id, created_at DESC);
```

**Rule 3: Equality Before Range**
```sql
-- Query: WHERE status = 'ACTIVE' AND created_at > '2024-01-01'
CREATE INDEX idx ON orders(status, created_at);
-- Equality on status first, then range on created_at
```

**Rule 4: Covering Index for Hot Queries**
```sql
-- Query: SELECT customer_id, amount FROM orders WHERE status = ?
CREATE INDEX idx_covering ON orders(status, customer_id, amount);
-- All needed columns in index — no back to table needed
```

### Composite Index Column Order — Real Example

```sql
-- Query 1
SELECT * FROM orders WHERE customer_id = 123 AND status = 'PENDING';
-- Best index: (customer_id, status) or (customer_id, status, ...)

-- Query 2
SELECT * FROM orders WHERE status = 'PENDING' AND created_at > '2024-01-01';
-- Best index: (status, created_at)

-- Query 3
SELECT * FROM orders WHERE customer_id = 123 ORDER BY created_at DESC;
-- Best index: (customer_id, created_at DESC)

-- One index can serve multiple queries if leftmost prefix matches
-- (customer_id, status, created_at) serves:
--   WHERE customer_id = ?
--   WHERE customer_id = ? AND status = ?
--   WHERE customer_id = ? AND status = ? ORDER BY created_at
```

### Other Index Types

| Type | Use Case |
|------|----------|
| **B-Tree** | Equality, range, sorting (default) |
| **Hash** | Equality only (rare in OLTP) |
| **GIN** | Full-text search, JSONB, arrays |
| **Bitmap** | Low cardinality columns in DW |
| **Partial** | `WHERE status = 'ACTIVE'` subset |
| **Functional** | `INDEX (LOWER(email))` for case-insensitive search |

### Index Trade-offs

**Pros:**
- Faster reads (O(log n) vs O(n))
- Faster sorts if index matches ORDER BY

**Cons:**
- Slower writes (every INSERT/UPDATE/DELETE updates indexes)
- Storage overhead
- Maintenance cost (REINDEX, fragmentation)

### When NOT to Index

- Small tables (full scan is fast)
- Low cardinality columns (gender, boolean)
- Frequently updated columns (overhead > benefit)
- Wide columns (large index size)

---

## Q18. How Do You Design Database Schemas to Handle High-Volume Transactional Data?

### Banking Example: 100M+ Transactions/Day

### Design Principles

**1. Normalization vs Denormalization**
- **OLTP** (transactional) → **3NF** (normalized)
- **OLAP** (analytical) → **denormalized** (star schema)
- For hybrid: normalize for writes, denormalize for reads (CQRS)

**2. Proper Data Types**
```sql
-- Don't use BIGINT when INT suffices
amount DECIMAL(19, 4) NOT NULL,         -- exact money (no float!)
currency CHAR(3) NOT NULL,                -- ISO 4217
created_at TIMESTAMP(6) NOT NULL,         -- microsecond precision
status ENUM('PENDING', 'COMPLETED', ...),  -- ENUM is MySQL-specific; PostgreSQL uses CREATE TYPE + CHECK constraints
transaction_id CHAR(26) NOT NULL,         -- ULID (sortable + unique)
```

**3. Partitioning**
```sql
-- Range partition by date (PostgreSQL example)
CREATE TABLE transactions (
    id BIGINT NOT NULL,
    customer_id BIGINT NOT NULL,
    amount DECIMAL(19, 4) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE TABLE transactions_2024_q1 PARTITION OF transactions
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
-- Range is half-open: [FROM, TO) — TO is exclusive.
-- So this partition holds 2024-01-01..2024-03-31 (inclusive).
-- Next partition must start at '2024-04-01'.
-- Old partitions can be archived / dropped easily
```

**4. Sharding Strategy**
```
Shard by customer_id (consistent hashing):
  shard = hash(customer_id) % num_shards

Each shard is independent — scales horizontally
```

**5. Indexing Strategy**
```sql
-- Hot path: lookup by customer + date range
CREATE INDEX idx_customer_date ON transactions(customer_id, created_at DESC);

-- Uniqueness on business key
CREATE UNIQUE INDEX uk_txn_id ON transactions(transaction_id);

-- For audit / compliance queries
CREATE INDEX idx_status_date ON transactions(status, created_at)
    WHERE status IN ('FAILED', 'PENDING');
```

**6. Audit Columns**
```sql
created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
created_by BIGINT NOT NULL,
updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
updated_by BIGINT NOT NULL,
version INT NOT NULL DEFAULT 0,           -- optimistic locking
deleted_at TIMESTAMP NULL                  -- soft delete
```

**7. Referential Integrity**
```sql
-- Foreign keys with proper indexes
customer_id BIGINT NOT NULL,
CONSTRAINT fk_customer FOREIGN KEY (customer_id) 
    REFERENCES customers(id) ON DELETE RESTRICT,
INDEX idx_customer (customer_id)
```

**8. Use Sequences/UUIDs Carefully**
```sql
-- UUIDv7 or ULID — sortable, unique, distributed-friendly
transaction_id CHAR(26) NOT NULL,           -- ULID
-- Avoid UUIDv4 as primary key (random, causes B-tree fragmentation)
```

### High-Volume Patterns

**Outbox Pattern (for messaging)**
```sql
CREATE TABLE outbox (
    id BIGSERIAL PRIMARY KEY,
    aggregate_type VARCHAR(50),
    aggregate_id VARCHAR(50),
    event_type VARCHAR(50),
    payload JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP NULL
);
CREATE INDEX idx_unprocessed ON outbox(created_at) WHERE processed_at IS NULL;
```

**Append-Only Pattern**
```sql
-- No UPDATE/DELETE — append only
-- Enables efficient partitioning + compression
CREATE TABLE ledger_entries (
    id BIGSERIAL,
    account_id BIGINT NOT NULL,
    debit DECIMAL(19, 4) DEFAULT 0,
    credit DECIMAL(19, 4) DEFAULT 0,
    balance_after DECIMAL(19, 4) NOT NULL,
    transaction_id BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
```

**Idempotency Table**
```sql
CREATE TABLE processed_messages (
    message_id VARCHAR(64) PRIMARY KEY,
    processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Performance Considerations

| Technique | Benefit |
|-----------|---------|
| Connection pooling | Reuse DB connections |
| Read replicas | Scale reads horizontally |
| Query result caching | Avoid repeated DB hits |
| Async writes | Decouple request from write |
| Batch operations | Reduce round trips |
| Prepared statements | Plan caching, no SQL injection |
| Batch inserts | 10-100x faster than individual |

### Banking-Specific Patterns

**Double-Entry Ledger**
```sql
-- Every transaction = 2 entries (debit + credit)
-- Sum of all entries for transaction = 0
CREATE TABLE ledger_entries (
    id BIGSERIAL PRIMARY KEY,
    txn_id BIGINT NOT NULL,
    account_id BIGINT NOT NULL,
    amount DECIMAL(19, 4) NOT NULL,   -- signed
    entry_type VARCHAR(10) NOT NULL,   -- DEBIT/CREDIT
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CHECK (amount != 0)
);
```

**Balance Materialization**
```sql
-- Snapshot table for fast balance lookups
-- Updated via trigger or application logic
CREATE TABLE account_balances (
    account_id BIGINT PRIMARY KEY,
    balance DECIMAL(19, 4) NOT NULL,
    last_txn_id BIGINT,
    updated_at TIMESTAMP
);
```

---

## Q19. How Do You Ensure Data Consistency When Concurrent Debit and Credit Operations Occur?

### The Problem

```
Initial balance: $100

Thread A: Debit $80 → reads balance 100, computes 20, writes 20
Thread B: Credit $50 → reads balance 100, computes 150, writes 150

Final balance: $150 ❌ (should be $70)
This is a "Lost Update" — Thread A's update was lost!
```

### Solution 1: Pessimistic Locking (SELECT ... FOR UPDATE)

```java
@Transactional
public void transfer(Long fromAccount, Long toAccount, BigDecimal amount) {
    // Lock both accounts in deterministic order (avoid deadlock!)
    Long first = Math.min(fromAccount, toAccount);
    Long second = Math.max(fromAccount, toAccount);
    
    Account a1 = accountRepository.findByIdForUpdate(first);   // SELECT ... FOR UPDATE
    Account a2 = accountRepository.findByIdForUpdate(second);
    
    Account from = (fromAccount == first) ? a1 : a2;
    Account to = (toAccount == first) ? a1 : a2;
    
    if (from.getBalance().compareTo(amount) < 0) {
        throw new InsufficientFundsException();
    }
    
    from.debit(amount);
    to.credit(amount);
    // Commits at end of transaction
}
```

**Pros:** Strong consistency, simple to reason about  
**Cons:** Blocking — reduces throughput under contention

### Solution 2: Optimistic Locking (Version Field)

```sql
CREATE TABLE accounts (
    id BIGINT PRIMARY KEY,
    balance DECIMAL(19, 4) NOT NULL,
    version INT NOT NULL DEFAULT 0
);
```

```java
@Entity
public class Account {
    @Id Long id;
    BigDecimal balance;
    
    @Version
    int version;   // JPA manages this
}

@Transactional
public void debit(Long accountId, BigDecimal amount) {
    Account account = accountRepository.findById(accountId).orElseThrow();
    account.debit(amount);   // in-memory update
    accountRepository.save(account);
    // UPDATE accounts SET balance = ?, version = version + 1 
    // WHERE id = ? AND version = ?
    // If 0 rows updated → OptimisticLockException
}
```

```java
@Transactional
public void debitWithRetry(Long accountId, BigDecimal amount) {
    int maxRetries = 3;
    for (int i = 0; i < maxRetries; i++) {
        try {
            doDebit(accountId, amount);
            return;
        } catch (OptimisticLockingFailureException e) {
            if (i == maxRetries - 1) throw e;
            Thread.sleep(10 * (i + 1));   // backoff
        }
    }
}
```

**Pros:** No locking overhead, scales well  
**Cons:** Retry logic needed, can fail under high contention

### Solution 3: Atomic SQL UPDATE (Single Statement)

```sql
-- Conditional update — atomic at DB level
UPDATE accounts
SET balance = balance - :amount
WHERE id = :accountId
  AND balance >= :amount;       -- prevent overdraft atomically

-- Check rows affected
-- If 0 rows → insufficient funds
```

```java
@Transactional
public boolean debit(Long accountId, BigDecimal amount) {
    int rowsAffected = jdbcTemplate.update(
        "UPDATE accounts SET balance = balance - ? WHERE id = ? AND balance >= ?",
        amount, accountId, amount
    );
    return rowsAffected == 1;
}
```

**Pros:** No lock contention, single round trip, atomic  
**Cons:** Cannot combine multiple operations easily

### Solution 4: Serializable Isolation Level

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void transfer(...) {
    // DB will detect conflicts and retry
}
```

**Pros:** Strongest guarantee  
**Cons:** Performance overhead, serialization failures → retries

### Solution 5: Compare-And-Swap (CAS)

```sql
UPDATE accounts
SET balance = :newBalance, version = version + 1
WHERE id = :accountId AND balance = :expectedBalance AND version = :expectedVersion;
```

```java
public boolean debit(Long accountId, BigDecimal amount) {
    Account account = accountRepository.findById(accountId).orElseThrow();
    BigDecimal expected = account.getBalance();
    BigDecimal newBalance = expected.subtract(amount);
    int version = account.getVersion();
    
    int rows = jdbcTemplate.update(
        "UPDATE accounts SET balance = ?, version = version + 1 " +
        "WHERE id = ? AND balance = ? AND version = ?",
        newBalance, accountId, expected, version
    );
    
    if (rows == 0) {
        throw new ConcurrentModificationException("Retry needed");
    }
    return true;
}
```

### Comparison Table

| Approach | Throughput | Complexity | Best For |
|----------|------------|------------|----------|
| Pessimistic Lock | Low | Simple | Low contention, critical operations |
| Optimistic Lock | High | Medium | Most web apps |
| Atomic UPDATE | Highest | Simple | Single-row updates |
| Serializable | Low | Medium | Complex multi-row transactions |
| CAS | High | Medium | High-contention single ops |

### Banking Best Practice

For banking, use a **hybrid approach**:

```java
@Transactional
public void transfer(Long from, Long to, BigDecimal amount) {
    // 1. Use atomic UPDATE for individual balance changes
    int debited = jdbcTemplate.update(
        "UPDATE accounts SET balance = balance - ? WHERE id = ? AND balance >= ? AND version = ?",
        amount, from, amount, expectedFromVersion
    );
    if (debited == 0) throw new InsufficientFundsException();
    
    int credited = jdbcTemplate.update(
        "UPDATE accounts SET balance = balance + ? WHERE id = ? AND version = ?",
        amount, to, expectedToVersion
    );
    if (credited == 0) {
        // Compensate — reverse the debit
        jdbcTemplate.update(
            "UPDATE accounts SET balance = balance + ? WHERE id = ?",
            amount, from
        );
        throw new ConcurrentModificationException();
    }
    
    // 2. Append ledger entry (append-only, no conflicts)
    ledgerRepository.save(new LedgerEntry(from, to, amount));
}
```

### Key Takeaways

- **Lost updates** are the #1 concurrency bug — always handle them
- **Always lock accounts in same order** to prevent deadlock
- **Idempotency** prevents double-application of the same operation
- **Append-only ledger** is your audit trail and conflict-free history
- **Atomic SQL** > application-level locking for single-row updates

---

## Production Topic: Database Indexing Mistakes

### Mistake 1: Too Many Indexes → Writes Slow Down

```sql
-- BAD: Index on every column
CREATE INDEX idx_email ON users(email);
CREATE INDEX idx_name ON users(name);
CREATE INDEX idx_phone ON users(phone);
CREATE INDEX idx_address ON users(address);
CREATE INDEX idx_created ON users(created_at);

-- Every INSERT/UPDATE/DELETE must update ALL indexes
-- Write performance degrades significantly
-- Storage overhead increases
```

**Rule of Thumb**: Only index columns that are frequently used in:
- `WHERE` clauses
- `JOIN` conditions
- `ORDER BY` clauses

**Monitor unused indexes**:
```sql
-- PostgreSQL
SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;

-- MySQL
SELECT * FROM sys.schema_unused_indexes;
```

### Mistake 2: Composite Index Column Order Matters

```sql
-- Query: WHERE customer_id = ? AND status = 'PENDING'
-- Best index: (customer_id, status)

-- BAD: Wrong order
CREATE INDEX idx_bad ON orders(status, customer_id);
-- This index CANNOT be used for the query above!
-- Because status has low selectivity (only 3 values)

-- GOOD: Correct order
CREATE INDEX idx_good ON orders(customer_id, status);
-- customer_id is first (high selectivity)
-- status is second (used for filtering after customer_id)
```

**Leftmost Prefix Rule**: A composite index `(a, b, c)` can be used for:
- `WHERE a = ?` ✅
- `WHERE a = ? AND b = ?` ✅
- `WHERE a = ? AND b = ? AND c = ?` ✅
- `WHERE b = ?` ❌ (cannot use index)
- `WHERE c = ?` ❌ (cannot use index)

### Mistake 3: No EXPLAIN ANALYZE Before Deployment → Production Incident Waiting to Happen

```sql
-- BAD: Deploy index without verifying it's used
CREATE INDEX idx_email ON users(email);
-- Hope it helps... but what if the query uses a function?

-- GOOD: Always verify with EXPLAIN ANALYZE
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user@example.com';

-- Look for:
-- - Index Scan (good) vs Seq Scan (bad for large tables)
-- - Actual rows vs Estimated rows (big difference = bad stats)
-- - Execution time
```

**What to Look For**:
```sql
-- GOOD: Index is being used
Index Scan using idx_email on users (cost=0.29..8.30 rows=1 width=100) (actual time=0.02..0.02 rows=1 loops=1)

-- BAD: Full table scan (index not used)
Seq Scan on users (cost=0.00..1550.00 rows=50000 width=100) (actual time=0.01..50.00 rows=50000 loops=1)
```

### Common Indexing Anti-Patterns

| Anti-Pattern | Problem | Fix |
|--------------|---------|-----|
| Index on low-cardinality column | Index not selective (e.g., `gender`, `boolean`) | Don't index, or use partial index |
| Index on frequently updated column | Write overhead > read benefit | Avoid or use covering index |
| Too many indexes | Slow writes, high storage | Remove unused indexes |
| Wrong column order | Index not used for queries | Order by selectivity |
| Missing `WHERE` in partial index | Index includes all rows | Add `WHERE` clause |
| Function in `WHERE` without function index | Index not used | Create function index |

### Partial Indexes

```sql
-- Only index active users (smaller, faster)
CREATE INDEX idx_active_users ON users(email) WHERE status = 'ACTIVE';

-- Only index pending orders
CREATE INDEX idx_pending_orders ON orders(created_at) WHERE status = 'PENDING';
```

### Function Indexes

```sql
-- Query uses LOWER() — regular index won't help
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- Create function index
CREATE INDEX idx_email_lower ON users(LOWER(email));
-- Now the query uses the index
```

### Key Takeaways

- **Too many indexes → writes slow down** — only index what you need
- **Composite index column order matters** — high selectivity first, equality before range
- **No EXPLAIN ANALYZE before deployment → production incident waiting to happen** — always verify indexes are used
- **Monitor unused indexes** and drop them
- **Use partial indexes** to reduce index size for filtered queries
- **Use function indexes** when queries use functions on indexed columns
