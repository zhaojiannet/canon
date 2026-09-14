---
name: postgres
description: Enforce PostgreSQL schema design and safe migrations. Use when editing **/migrations/*.sql, **/schema/*.sql, **/migrate/*.sql, or when the user mentions CREATE TABLE, ALTER TABLE, foreign key, index, jsonb, BIGSERIAL, timestamptz, migration, NOT NULL, DROP TABLE, or CREATE INDEX. Keeps every table on one shape (snake_case, timestamptz, FK plus index, JSONB), wraps migrations in a transaction, builds indexes CONCURRENTLY, adds NOT NULL columns in steps, and stops for approval before any destructive op.
paths:
  - "**/migrations/*.sql"
  - "**/migrations/**/*.sql"
  - "**/schema/*.sql"
  - "**/schema/**/*.sql"
  - "**/migrate/*.sql"
  - "**/migrate/**/*.sql"
  - "**/sqitch/**/*.sql"
---

> Targets PostgreSQL 18 · verified 2026-09 (latest 18.6). The DDL rules below also hold on 14–17, which upstream still supports.

This skill covers two sides of PostgreSQL DDL: **schema design** (naming and shape) and **migration safety** (change-safety). The rules: tables follow a consistent shape, and every migration runs on a live production database without downtime. If a change cannot, **STOP** and ask first. For one-off data fixes not bound for production schema, say so explicitly so this skill relaxes.

# Schema design

## Naming

- **Table names**: `snake_case`, plural noun. `users`, `order_items`. Not `User`, `OrderItem`, `tbl_user`.
- **Column names**: `snake_case`. `created_at`, `email_address`, `is_active`. No camelCase.
- **Primary key**: always `id`. `BIGSERIAL` for auto-increment, `UUID` (with `gen_random_uuid()` default) for distributed/external-facing. (The SQL-standard identity column and PG 18's time-ordered `uuidv7()` also exist; this skill keeps the choices above.)
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

- `NO ACTION` (default when `ON DELETE` is omitted) — deleting a referenced parent fails if children still exist, but the check can be deferred to later in the transaction
- `RESTRICT` — stricter than `NO ACTION`: prevents deleting a referenced parent, and the check cannot be deferred
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

- **Always wrap in a transaction**: `BEGIN; ...; COMMIT;`. A failed step rolls back. Some statements cannot run inside a transaction block (`CREATE INDEX CONCURRENTLY`); put these in their own migration file. `ALTER TYPE ... ADD VALUE` can run inside a transaction, but the new enum value cannot be used until that transaction commits; use it in a later migration.
- **Idempotent guards**: `IF EXISTS` on `DROP`, `IF NOT EXISTS` on `CREATE`. A migration that fails halfway should be safe to retry.
- **No long locks during business hours**: `ALTER TABLE` that rewrites the entire table (adding a column with a default before PG 11, adding a column with a volatile default such as `clock_timestamp()`, changing column type) takes an `AccessExclusiveLock` and blocks reads.
- **Never lose data without explicit approval**: `DROP TABLE`, `DROP COLUMN`, `TRUNCATE`, `DELETE` without `WHERE`, removing an enum value (there is no `DROP VALUE`; it takes dropping and re-creating the enum type) are destructive. STOP and ask the user before writing them.

## Exact patterns

Adding a `NOT NULL` column, renaming a column, or creating an index on a live table each need a specific multi-step sequence — read [references/migrations.md](references/migrations.md) before writing any of them, plus the full list of forbidden patterns. Writing these from memory is how a production table gets rewritten under an `AccessExclusiveLock`.

## When the change is destructive

`DROP TABLE`, `DROP COLUMN`, `TRUNCATE`, `DELETE` without `WHERE`, dropping and re-creating an enum type to remove a value: these can lose data. **STOP** and report:

> Migration [filename] contains [destructive op]. Confirm:
> 1. The data is no longer needed (or has been backed up).
> 2. No production code reads the column/table.
> 3. The change is rolled out after the code that stops using it.
> Approve to proceed?

# Verification (grep after every schema/migration change)

```bash
# SQL files under this skill's paths
sqlfiles() { find . -type f -name '*.sql' \( -path '*/migrations/*' -o -path '*/schema/*' -o -path '*/migrate/*' -o -path '*/sqitch/*' \) -not -path '*/node_modules/*' -print0; }
sqlgrep() { sqlfiles | xargs -0 grep -HnE "$@"; }

# schema shape
sqlgrep -i '\btimestamp\b' | grep -viE 'timestamptz|\btimestamp\s*(\([0-9]+\))?\s+with\s+time\s+zone'   # should be timestamptz
sqlgrep -i '\b(smallserial|serial|serial2|serial4)\b'                                                    # should be BIGSERIAL
sqlgrep -i '\bjson\b\s*($|[,;)]|\[|not\b|null\b|default\b|check\b|using\b|collate\b)'                     # should be JSONB
sqlgrep '\b[a-z]+[A-Z][a-zA-Z]*\b'                                                                       # camelCase
sqlgrep -i '\b(tbl|col)_'                                                                                # table/column prefixes

# migration safety
sqlgrep -i '\bdrop\s+table\b'                                                                            # destructive, approval needed even with IF EXISTS
sqlgrep -i '^\s*truncate\b'
sqlgrep -i '\bdelete\s+from\s+[a-z0-9_."]+\s*;'                                                          # DELETE without WHERE
sqlgrep -i '\badd\s+column\s+(if\s+not\s+exists\s+)?[a-z0-9_"]+\s[^;]*\bnot\s+null\b' | grep -viE '\bdefault\b'   # NOT NULL without default
sqlgrep -i '^\s*create\s+(unique\s+)?index\s' | grep -viE '\bconcurrently\b' | while IFS=: read -r f n line; do   # blocking index on a table this file did not create
  t=$(printf '%s\n' "$line" | sed -nE 's/.*[[:space:]][oO][nN][[:space:]]+([oO][nN][lL][yY][[:space:]]+)?([a-zA-Z0-9_."]+).*/\2/p')
  [ -n "$t" ] && grep -qiE "create\s+table\s+(if\s+not\s+exists\s+)?$t([^a-z0-9_]|$)" "$f" || echo "$f:$n:$line"
done
sqlfiles | xargs -0 grep -LiE --null '^\s*(begin(\s+(transaction|work))?|start\s+transaction)(\s+isolation\s+level\s+[a-z ]+)?\s*;' | xargs -0 grep -LiE '\bconcurrently\b' | grep -v '/schema/'   # migration without BEGIN (CONCURRENTLY files and schema/ exempt)
```

Reference: https://www.postgresql.org/docs/current/ddl.html · https://www.postgresql.org/docs/current/sql-altertable.html
