# Relational Database & ORM

## 1. Design tables for tickets, comments, and audit logs

**Entities:**
- `tickets`: id (PK), external_id, title, description, status, priority, assignee_id, created_at, updated_at, data_classification
- `comments`: id (PK), ticket_id (FK), author_id, body, is_internal, created_at
- `audit_logs`: id (PK), entity_type, entity_id, action, actor_id, before_json, after_json, correlation_id, created_at

**Indexes:** `(ticket_id, created_at)` on comments; `(entity_type, entity_id)` on audit_logs; `external_id` unique on tickets.

**Normalization:** Separate lookups for status, priority, classification if shared.

## 2. ORM vs raw SQL — when to bypass the ORM

**Use ORM for:** CRUD, simple filters, joins within relationship graph, migrations — keeps code readable, portable, testable.

**Bypass for:** Complex aggregations, window functions, bulk upserts, recursive CTEs, verified hot paths where ORM generates N+1 or poor plans.

**Rule:** Profile first. If bypassing, add integration test that asserts SQL shape and result correctness. Document why.

## 3. Database transactions in an AI workflow

**Scenario:** Retrieve ticket → fetch context → call model → write summary + audit record.

**Transaction boundary:** Start after validation, commit after summary and audit are written. If model call fails, rollback both.

**Isolation:** `READ_COMMITTED` default; use `REPEATABLE_READ` or `SERIALIZABLE` only if phantom reads cause correctness bugs (rare).

**Keep short:** Do not hold transaction across external HTTP calls. Do the model call outside, then re-acquire for the write phase, or use saga/outbox pattern.

## 4. Schema migrations safely

- Add column: nullable, default if needed → deploy code → backfill → add NOT NULL if required.
- Rename: add new column → dual-write → backfill → switch reads → drop old.
- Delete: deprecate in code first → verify no reads → drop column in separate deploy.
- Always run migrations in a transaction (Django does this; Alembic can).
- Measure migration time against maintenance window; use `pg_repack` or online tools for large tables.

## 5. Handling concurrent updates to the same ticket

**Optimistic locking:** Add `version` column; `UPDATE ... WHERE id=? AND version=?`; retry on 0 rows affected.

**Pessimistic locking:** `SELECT ... FOR UPDATE` — only for short critical sections.

**Idempotency:** For retried webhook/async job, use idempotency key stored in a unique index; return existing result on duplicate.

**Trade-off:** Optimistic scales better; pessimistic simpler for very high contention.