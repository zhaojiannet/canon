---
name: pg-schema
description: Enforce PostgreSQL schema design conventions. Use this skill when editing **/migrations/*.sql, **/schema/*.sql, or when the user mentions table, column, primary key, foreign key, index, constraint, JSON column, jsonb, schema, ddl, CREATE TABLE, BIGSERIAL, UUID, timestamp. Forbids ad-hoc naming, missing FK constraints, SERIAL when BIGSERIAL works, missing timestamps, jsonb for structured first-class data, timestamp without time zone.
paths:
  - "**/migrations/*.sql"
  - "**/migrations/**/*.sql"
  - "**/schema/*.sql"
  - "**/schema/**/*.sql"
allowed-tools:
  - Read
  - Grep
---

This skill enforces PostgreSQL schema design conventions. Pair this with `postgresql-migration-safety`—that skill covers change-safety; this skill covers naming and shape.

Apply when authoring `CREATE TABLE` / `ALTER TABLE ... ADD CONSTRAINT` in migrations.

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

## Forbidden patterns

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

## Verification (grep after every schema/migration change)

```bash
grep -rnE '\btimestamp\b' --include='*.sql' migrations/ schema/ 2>/dev/null | grep -vE 'timestamptz|timestamp with time zone'  # should be timestamptz
grep -rnE 'SERIAL\s+PRIMARY' --include='*.sql' migrations/ 2>/dev/null                       # should be BIGSERIAL
grep -rnE '\bJSON\s+(NOT NULL|DEFAULT|,|$)' --include='*.sql' migrations/ 2>/dev/null         # should be JSONB
grep -rnE '\b[a-z]+[A-Z][a-zA-Z]*\b' --include='*.sql' migrations/ 2>/dev/null                # camelCase
grep -rnE 'tbl_|col_' --include='*.sql' migrations/ 2>/dev/null                                # table/column prefixes
```

Reference: https://www.postgresql.org/docs/current/ddl.html
