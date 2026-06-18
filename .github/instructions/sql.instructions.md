---
description: "Use when writing SQL or database migrations. Covers parameterized queries, migration safety, indexing, connection pooling, and query performance."
applyTo: "**/*.sql"
---

# SQL & Database Conventions

## Parameterized Queries — Mandatory

Never concatenate user input into SQL strings. This is the #1 security vulnerability.

```sql
-- Bad — SQL injection
SELECT * FROM users WHERE email = '${email}';

-- Good — parameterized (placeholder syntax varies by driver)
SELECT * FROM users WHERE email = ?;
```

- Use parameterized queries everywhere: `?` (MySQL/SQLite), `$1` (Postgres), `:name` (named params)
- ORMs: ensure the ORM parameterizes. Raw queries still need placeholders.
- **No dynamic table/column/group BY names from user input** — use allowlists for these
- Stored procedures: use parameterized calls, never `EXEC` with string concatenation

## Migration Safety — Mandatory

Every migration must be reversible and non-destructive.

```sql
-- Migration: add column (safe, non-blocking)
ALTER TABLE users ADD COLUMN preferences JSONB DEFAULT '{}' NOT NULL;

-- Migration: drop column (DANGEROUS — requires multi-step)
-- Step 1: Stop writing to column in application code. Deploy.
-- Step 2: Migration to drop column (only after confirming no reads/writes).
```

- **Never drop a column or table in the same migration that adds it** — split into separate deploys
- **Never rename a column**: the old name still exists in running application code. Add new column, dual-write, migrate data, remove old column.
- **Backfill large tables in batches**, not in a single transaction:

```sql
-- Bad — locks the table for minutes
UPDATE users SET status = 'active' WHERE status IS NULL;

-- Good — batch processing
UPDATE users SET status = 'active'
WHERE id IN (SELECT id FROM users WHERE status IS NULL LIMIT 1000);
-- Repeat until no rows affected
```

- **Always add a default value** for new NOT NULL columns
- **Test rollback**: every `up` migration must have a tested `down` migration
- **Use advisory locks** for migrations that shouldn't run concurrently across multiple replicas

## Indexing

Indexes are free to read but expensive to write. Be intentional.

```sql
-- Covering index for common query pattern
CREATE INDEX idx_users_email_status ON users (email, status);

-- Partial index — only indexes rows matching condition (smaller, faster)
CREATE INDEX idx_orders_pending ON orders (created_at)
    WHERE status = 'pending';

-- Index on expression (Postgres)
CREATE INDEX idx_users_lower_email ON users (LOWER(email));
```

- **Index columns used in WHERE, JOIN, ORDER BY** — every such column should be indexed or justified
- **Multi-column index column order matters**: most selective columns first. Index on `(a, b)` covers queries on `(a)` and `(a, b)` but NOT `(b)` alone
- **Check the query plan**: `EXPLAIN ANALYZE` before and after adding indexes
- **Don't over-index**: every index slows down INSERT/UPDATE/DELETE. Index what you query, not everything.
- **Remove unused indexes**: they waste disk space and write performance

## Connection Pooling

Never create a new connection per request.

- **Use the framework's connection pool**: `pgbouncer`, `HikariCP`, `sqlx::PgPool`, SQLAlchemy `QueuePool`
- **Pool size**: start with `(2 * CPU cores) + 1` for active connections. Tune from there, never default to 100.
- **Connection timeout**: 30 seconds max. A stuck connection holds a pool slot and degrades everything.
- **Statement timeout**: set at the pool level to prevent runaway queries:

```sql
-- Postgres: abort any query that runs longer than 30s
SET statement_timeout = '30s';
```

- **Never leak connections**: use `try-with-resources`, `defer`, context managers, or async pools that auto-return

## Transaction Boundaries

Every write that touches multiple rows or tables needs a transaction.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

- **Wrap related writes in transactions** — partial writes corrupt data
- **Keep transactions short**: no API calls, no file I/O, no user input inside a transaction
- **Use the right isolation level**: `READ COMMITTED` (default, good enough), `REPEATABLE READ` (for consistent reads), `SERIALIZABLE` (when correctness matters most). Understand the tradeoffs.
- **Handle rollback gracefully**: catch errors, rollback, return a meaningful error — don't leave the connection in a broken state

## N+1 Prevention

One query that returns 100 rows beats 100 queries that return 1 row.

```sql
-- Bad — N+1: one query for users, then one query per user for posts
SELECT * FROM users;                          -- 1 query
SELECT * FROM posts WHERE user_id = 1;        -- N queries
SELECT * FROM posts WHERE user_id = 2;        -- ...

-- Good — one query with JOIN
SELECT u.*, p.* FROM users u
LEFT JOIN posts p ON p.user_id = u.id;
```

- **JOIN instead of loop**: if you need related data, fetch it in one query
- **IN clause for batches**: `SELECT * FROM posts WHERE user_id IN (1, 2, 3, ...)` when you have a set of IDs
- **Eager loading**: ORMs have this built in — use it
- **Lazy loading is a trap**: it's convenient and always wrong at scale

## Query Performance

- **Never `SELECT *` in production code**: return only the columns you need. `SELECT *` breaks when columns are added, wastes bandwidth, and prevents index-only scans.
- **Avoid functions on indexed columns in WHERE**: `WHERE LOWER(email) = 'x'` can't use a plain index on `email`. Either index the expression or normalize the input.
- **`LIKE '%prefix'` can't use an index**: the leading wildcard makes it a full scan. Use full-text search or trigram indexes instead.
- **`LIMIT` without `ORDER BY` is non-deterministic**: results may change between pages
- **`OFFSET` is slow for deep pages**: each page re-scans all previous rows. Use cursor-based pagination for large datasets.

## Schema Design

- **Use UUIDs or ULIDs for primary keys** on user-facing entities (prevents enumeration, works for distributed systems). Auto-increment integers are fine for internal-only tables.
- **Choose the right data type**: `TEXT` not `VARCHAR(255)` (no performance difference in Postgres, no arbitrary limit). `BIGINT` for IDs that will grow. `TIMESTAMPTZ` (with timezone!) not `TIMESTAMP`.
- **Add `created_at` and `updated_at` to every table**: `updated_at` auto-updates with triggers or ORM hooks.
- **Normalize by default**: no JSON blobs for structured data. Denormalize only when you have a measured performance problem.
- **Foreign keys**: always declare them. They document relationships and prevent orphan rows. Index FK columns — they're used in JOINs.
