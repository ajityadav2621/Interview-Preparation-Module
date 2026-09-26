# Database Indexes Internally

## How Indexes Work

### B-Tree Structure
- Self-balancing tree structure
- Sorted data for efficient range queries
- O(log n) search complexity

### Index Components
```sql
CREATE INDEX idx_user_email ON users(email);
-- Creates B-tree on email column
-- Stores email values with row pointers
```

## Index Types

### 1. **Single Column Index**
```sql
CREATE INDEX idx_email ON users(email);
```

### 2. **Composite Index**
```sql
-- Order matters! Most selective column first
CREATE INDEX idx_last_first ON users(last_name, first_name);
```

### 3. **Unique Index**
```sql
CREATE UNIQUE INDEX idx_username ON users(username);
```

## Index Selection Rules

### Column Order in Composite Index
- **Most selective column first** - reduces index size
- **Equality conditions first** - then range conditions
- **Prefix matching** - leftmost prefix rule

```sql
-- Good: equality then range
CREATE INDEX idx_status_date ON orders(status, created_date);

-- Query uses index efficiently
SELECT * FROM orders WHERE status = 'PENDING' AND created_date > '2024-01-01';

-- Bad: range before equality
CREATE INDEX idx_date_status ON orders(created_date, status);
-- Same query won't use index efficiently
```

## When to Use Indexes

### Good Candidates
- **Frequently queried columns** (WHERE clauses)
- **Join columns** (foreign keys)
- **Order by columns**
- **High cardinality columns** (many unique values)

### Avoid Over-Indexing
- Each index adds write overhead
- More indexes = slower inserts/updates/deletes
- Storage space increases

## Interview Tip
Explain that indexes are a trade-off between read and write performance. Always analyze query patterns before creating indexes.