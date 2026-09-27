# Module 3: Databases — PostgreSQL/MySQL, SQL Optimization & Multi-Tenancy

---

## PART 1: Fundamentals (must know cold)

- **Primary keys, foreign keys, normalization** at a conceptual level (1NF–3NF) — know why normalization reduces redundancy and what denormalization trades away (redundancy/storage) for (fewer joins, faster reads).
- **Indexes**: a B-tree index lets the DB find rows in O(log n) instead of scanning the whole table. Indexes speed up reads but slow down writes (every insert/update must also update the index) and use disk space — don't index everything blindly.
- **JOIN types**: INNER (matches only), LEFT (all of left + matches or NULL), RIGHT, FULL OUTER — know when each is correct.
- **Transactions & ACID**: Atomicity, Consistency, Isolation, Durability — a transaction either fully commits or fully rolls back; isolation levels control how much one transaction can see of another's uncommitted changes.
- **N+1 query problem**: looping and issuing one query per row instead of a single batched/joined query — the single most common real-world performance bug.
- **`EXPLAIN ANALYZE`**: the tool for seeing a query's actual execution plan — whether it's using an index (index scan) or scanning the whole table (sequential scan).
- **Multi-tenancy patterns**: (1) shared DB + shared schema with a `tenant_id` column on every table (cheapest, most common for SaaS), (2) shared DB + schema-per-tenant, (3) database-per-tenant (most isolated, most operational overhead).
- **Connection pooling**: reusing a fixed set of DB connections across requests instead of opening a new one per request, since connection setup has real overhead.

---

## PART 2: Interview Questions & Answers

**Q1: Walk me through your process for finding and fixing a slow query.**
> I'd run `EXPLAIN ANALYZE` on the query to see the actual execution plan — specifically checking for a sequential scan on a large table where an index should apply. If it's a multi-tenant query, I'd check that `tenant_id` is the leading column of the relevant composite index, since every query filters by tenant first. If it's an ORM-generated query, I'd check for the N+1 pattern — one query per row in a loop — and replace it with a single JOIN or `IN` clause. After the fix, I re-run `EXPLAIN ANALYZE` to confirm the plan actually changed, not just assume the fix worked.

**Q2: What's the N+1 query problem, and how do you spot it in production?**
> It's when code loops over a set of records and issues a separate query per record to fetch related data, instead of one query that fetches everything with a JOIN. It's easy to introduce accidentally with an ORM — accessing a lazy-loaded relationship inside a loop is the classic trigger. I'd spot it by looking at query logs/APM traces for a suspiciously high query count relative to the number of result rows returned, or by profiling a slow endpoint and seeing dozens of near-identical queries fire in sequence.

**Q3: How would you design tenant isolation for a multi-tenant SaaS so one tenant can never see another's data?**
> Every tenant-owned table gets a non-nullable `tenant_id` column, and every query filters by it — ideally enforced at the data-access layer (a base repository that automatically injects the tenant filter) rather than trusting every developer to remember it manually in every query. I'd derive `tenant_id` server-side from the authenticated session, never from client input. As a backstop, PostgreSQL row-level security policies can enforce this at the database level even if application code has a bug.

**Q4: When would you choose schema-per-tenant or database-per-tenant instead of a shared schema with `tenant_id`?**
> Shared schema is the cheapest and simplest to operate and scales well for most SaaS use cases. I'd move to schema-per-tenant or database-per-tenant when a specific tenant needs stronger data isolation guarantees (e.g., a regulatory/compliance requirement), needs independent backup/restore, or has usage patterns heavy enough that noisy-neighbor performance impact on other tenants becomes a real problem — the trade-off is meaningfully higher operational complexity (migrations now have to run against every tenant's schema/DB).

**Q5: What's the difference between a clustered and non-clustered index, and does it matter in practice?**
> A clustered index determines the physical order rows are stored on disk (a table can only have one, since data can only be ordered one way); a non-clustered index is a separate structure pointing back to the row's location. In practice for MySQL/InnoDB, the primary key is the clustered index — so querying by primary key is fastest, and every secondary index internally stores the primary key to look up the full row, which is why an overly large primary key type can bloat every secondary index too.

---

## PART 3: How It Works Internally

**How a B-tree index actually speeds up a query**: the index is a balanced tree structure sorted by the indexed column(s). Instead of scanning every row (O(n)), the database walks down the tree comparing the search value at each node, reaching the target row's location in O(log n) comparisons. This is also why indexes only help queries that can use them from the left — a composite index on `(tenant_id, created_at)` speeds up queries filtering on `tenant_id` alone or on both columns together, but not a query filtering only on `created_at`.

**How the query planner decides a JOIN strategy**: PostgreSQL/MySQL's cost-based optimizer estimates the cost of different execution strategies — nested loop join (good for small tables or when one side is tiny), hash join (build a hash table of the smaller table, probe it with the larger — good for large unsorted tables), merge join (good when both inputs are already sorted on the join key) — using table statistics (row counts, data distribution) it maintains internally. This is why the same query can execute differently as data grows — the planner re-evaluates cost estimates, and a plan that was optimal at 10K rows might not be at 10M rows (a reason to periodically re-run `ANALYZE` so statistics stay current).

**How transactions provide ACID internally**: 
- **Atomicity/Durability**: via write-ahead logging (WAL) — changes are written to a log *before* being applied to the actual data files, so if the system crashes mid-transaction, recovery replays or discards the log to restore a consistent state.
- **Isolation**: via locking or MVCC (multi-version concurrency control) — PostgreSQL uses MVCC, meaning a transaction sees a consistent snapshot of the data as of when it started, even while other transactions are concurrently writing; readers don't block writers and vice versa, which is why Postgres handles concurrent read-heavy + write-heavy workloads well without much explicit locking in application code.

**How connection pooling works internally**: a pool (e.g., PgBouncer, or a library-level pool) maintains a fixed number of already-open DB connections. When your app needs to run a query, it borrows a connection from the pool, uses it, and returns it — rather than the expensive process of establishing a new TCP connection + auth handshake with the database on every single request. Under load, if all pooled connections are busy, new requests queue briefly rather than the database being overwhelmed by unbounded new connections.
