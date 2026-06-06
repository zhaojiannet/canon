---
name: postgres
description: Enforce PostgreSQL schema design and safe migrations. Use when editing **/migrations/*.sql, **/schema/*.sql, **/migrate/*.sql, or when the user mentions CREATE TABLE, ALTER TABLE, foreign key, index, jsonb, BIGSERIAL, timestamptz, migration, NOT NULL, DROP TABLE, or CREATE INDEX. Forbids ad-hoc naming, missing FK/index, plain timestamp, JSON over JSONB, destructive ops without approval, NOT NULL adds without a default, blocking index builds, and transaction-less migrations.
paths:
  - "**/migrations/*.sql"
  - "**/migrations/**/*.sql"
  - "**/schema/*.sql"
  - "**/schema/**/*.sql"
  - "**/migrate/*.sql"
  - "**/migrate/**/*.sql"
  - "**/sqitch/**/*.sql"
allowed-tools:
  - Read
  - Grep
---

This skill covers two sides of PostgreSQL DDL: **schema design** (naming and shape) and **migration safety** (change-safety). The rules: tables follow a consistent shape, and every migration runs on a live production database without downtime. If a change cannot, **STOP** and ask first. For one-off data fixes not bound for production schema, say so explicitly so this skill relaxes.

# Schema design

## Naming

- **Table names**: `snake_case`, plural noun. `users`, `order_items`. Not `User`, `OrderItem`, `tbl_user`.
- **Column names**: `snake_case`. `created_at`, `email_address`, `is_active`. No camelCase.
- **Primary key**: always `id`. `BIGSERIAL` for auto-increment, `UUID` (with `gen_random_uuid()` default) for distributed/external-facing.
- **Foreign keys**: `<referenced_table_singular>_id`. `users.org_id` references `organizations.id`.
- **Boolean columns**: prefix with `is_` / `has_`. `is_active`, `has_paid`. Default a sensible value, not nullable.
- **Timestamp columns**: `created_at`, `updated_at`, `deleted_at` (if soft delete). All `timestamptz`, never `timestamp`. Default `now()`.
- **Index names**: `idx_<table>_<columns>`. `idx_users_org_id`, `idx_orders_user_id_created_at`.
- **Constraint names**: explicit, not auto-generated. `users_email_key` (unique), `users_email_check` (check), `orders_user_id_fkey` (foreign key).

## Required columns on every business table

```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  -- business columns
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
```

`updated_at` is maintained by a trigger or by application code consistently. Trigger version:

```sql
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS trigger AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_updated_at BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

## Foreign keys

Every `<table>_id` column referencing another table needs a foreign key with explicit `ON DELETE`:

```sql
ALTER TABLE orders ADD CONSTRAINT orders_user_id_fkey
  FOREIGN KEY (user_id) REFERENCES users(id)
  ON DELETE RESTRICT;
```

- `RESTRICT` (default) — cannot delete parent if children exist
- `CASCADE` — delete children with parent
- `SET NULL` — make children orphans (column must be nullable)

Each choice is a business decision. Do not omit.

## Indexes

- One index per query pattern. Do not pre-index every column.
- Composite indexes order matters: most-selective column first, then range columns.
- For lookups by foreign key: index the FK column. Postgres does not auto-index the referencing side.
- For text search: `tsvector` + GIN, not `LIKE '%foo%'` on a btree.
- For exact-match on `varchar`/`text`: btree default works.

## jsonb usage

`jsonb` is for **schemaless** data: user-supplied JSON, third-party API payloads, plugin config. **Not** for structured first-class data. If you need `WHERE foo->>'name' = 'x'`, you probably need a real column.

Allowed:
```sql
CREATE TABLE webhook_events (
  id BIGSERIAL PRIMARY KEY,
  source text NOT NULL,
  payload jsonb NOT NULL,        -- third-party shape, varies
  received_at timestamptz NOT NULL DEFAULT now()
);
```

Forbidden:
```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  data jsonb NOT NULL              -- ❌ should be split into concrete columns (email / name / phone)
);
```

## Forbidden schema patterns

- `SERIAL` PK on a table that may grow past 2 billion rows. Use `BIGSERIAL`.
- Mixing UUID and BIGSERIAL in the same project without explicit reason.
- `timestamp without time zone` (i.e. plain `timestamp`). Use `timestamptz` always.
- Nullable columns by default. Default to `NOT NULL` and add `DEFAULT` if needed.
- Foreign key column without an index.
- Unique constraint without an explicit name.
- `JSON` type. Use `JSONB`.
- camelCase column names like `userId`, `createdAt`.
- Tables/columns prefixed with `tbl_` / `col_`.
- A new column / index / constraint when the existing schema already defines an equivalent. grep `migrations/` and `schema/` first; reuse or extend if found.

## When the design is unconventional

If you need a non-standard pattern (composite primary key, partitioned table, materialized view, foreign data wrapper), **STOP** and report:

> Need [pattern] for [reason]. Standard pattern would be [...]. Approve [unconventional pattern] for [specific reason]?

# Migration safety

The rule: every migration runs on a live production database without downtime. If it cannot, **STOP** and ask first.

## Core principles

- **Always wrap in a transaction**: `BEGIN; ...; COMMIT;`. A failed step rolls back. Some migrations cannot run inside a transaction (`CREATE INDEX CONCURRENTLY`, `ALTER TYPE ... ADD VALUE`); put these in their own migration file.
- **Idempotent guards**: `IF EXISTS` on `DROP`, `IF NOT EXISTS` on `CREATE`. A migration that fails halfway should be safe to retry.
- **No long locks during business hours**: `ALTER TABLE` that rewrites the entire table (adding a `NOT NULL` column without a default before PG 11, changing column type) takes an `AccessExclusiveLock` and blocks reads.
- **Never lose data without explicit approval**: `DROP TABLE`, `DROP COLUMN`, `TRUNCATE`, `DELETE` without `WHERE`, `ALTER TYPE ... DROP VALUE` are destructive. STOP and ask the user before writing them.

## Forbidden migration patterns

- `DROP TABLE foo;` without `IF EXISTS` and without explicit user approval.
- `TRUNCATE foo;` without explicit user approval.
- `DELETE FROM foo;` without `WHERE`. This is functionally `TRUNCATE` and locks the table.
- `ALTER TABLE foo ADD COLUMN bar text NOT NULL;` (no default). On a non-empty table this fails. Either add with `DEFAULT` (PG 11+ is non-rewriting), or split into three migrations.
- `CREATE INDEX idx_foo ON foo(bar);` on a large production table. Use `CREATE INDEX CONCURRENTLY` in its own non-transactional migration.
- `ALTER TABLE foo RENAME COLUMN old TO new;` deployed atomically with code that reads from `new`. Old code still in flight breaks. Split: add new, dual-write, switch reads, drop old.
- `ALTER TABLE foo ALTER COLUMN bar TYPE int USING bar::int;` on a large table. Same rewrite problem.
- Migration without `BEGIN; ... COMMIT;` (unless the operation cannot run in a transaction).
- Mixing data backfill (UPDATE millions of rows) with schema change in one transaction. Long-running transactions hold locks and bloat WAL.
- A new migration when an unmerged migration in the same branch already changes the same table/column. grep `migrations/` first; consolidate if found.

## Safe pattern: add NOT NULL column

Three steps:

```sql
-- migration N: add nullable
ALTER TABLE users ADD COLUMN status text;

-- migration N+1: backfill (in batches if large)
UPDATE users SET status = 'active' WHERE status IS NULL;

-- migration N+2: set NOT NULL
ALTER TABLE users ALTER COLUMN status SET NOT NULL;
```

PG 11+ accepts a default for the add step that does not rewrite the table:

```sql
ALTER TABLE users ADD COLUMN status text NOT NULL DEFAULT 'active';
ALTER TABLE users ALTER COLUMN status DROP DEFAULT;  -- only if the default existed solely for the ADD step
```

## Safe pattern: rename column

```sql
-- migration N: add new column, keep both, code dual-writes
ALTER TABLE users ADD COLUMN email_address text;
UPDATE users SET email_address = email;

-- migration N+1: drop old (after deploy switches reads)
ALTER TABLE users DROP COLUMN email;
```

## Safe pattern: large index

```sql
-- separate file, no BEGIN/COMMIT
CREATE INDEX CONCURRENTLY idx_users_org_id ON users(org_id);
```

If the create is interrupted, the index is left invalid:
```sql
SELECT pg_class.relname FROM pg_class
JOIN pg_index ON pg_index.indexrelid = pg_class.oid
WHERE pg_index.indisvalid = false;
```
Then `DROP INDEX` and recreate.

## When the change is destructive

`DROP TABLE`, `DROP COLUMN`, `TRUNCATE`, `DELETE` without `WHERE`, `ALTER TYPE DROP VALUE`: these can lose data. **STOP** and report:

> Migration [filename] contains [destructive op]. Confirm:
> 1. The data is no longer needed (or has been backed up).
> 2. No production code reads the column/table.
> 3. The change is rolled out after the code that stops using it.
> Approve to proceed?

# Verification (grep after every schema/migration change)

```bash
# schema shape
grep -rnE '\btimestamp\b' --include='*.sql' migrations/ schema/ 2>/dev/null | grep -vE 'timestamptz|timestamp with time zone'  # should be timestamptz
grep -rnE 'SERIAL\s+PRIMARY' --include='*.sql' migrations/ 2>/dev/null                       # should be BIGSERIAL
grep -rnE '\bJSON\s+(NOT NULL|DEFAULT|,|$)' --include='*.sql' migrations/ 2>/dev/null         # should be JSONB
grep -rnE '\b[a-z]+[A-Z][a-zA-Z]*\b' --include='*.sql' migrations/ 2>/dev/null                # camelCase
grep -rnE 'tbl_|col_' --include='*.sql' migrations/ 2>/dev/null                                # table/column prefixes

# migration safety
grep -rnE 'DROP\s+TABLE' --include='*.sql' migrations/ 2>/dev/null | grep -vi 'IF EXISTS'
grep -rnE '^\s*TRUNCATE\b' --include='*.sql' migrations/ 2>/dev/null
grep -rnE 'DELETE\s+FROM\s+\w+\s*;' --include='*.sql' migrations/ 2>/dev/null                  # DELETE without WHERE
grep -rnE 'ADD\s+COLUMN\s+\w+\s+\w+\s+NOT\s+NULL\s*;' --include='*.sql' migrations/ 2>/dev/null # no default value
grep -rnE '^\s*CREATE\s+INDEX\s+' --include='*.sql' migrations/ 2>/dev/null | grep -v 'CONCURRENTLY'
grep -rL 'BEGIN' migrations/*.sql 2>/dev/null                                                   # missing transaction wrapper
```

Reference: https://www.postgresql.org/docs/current/ddl.html · https://www.postgresql.org/docs/current/sql-altertable.html
