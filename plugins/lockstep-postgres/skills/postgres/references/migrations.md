# Migration patterns

Exact multi-step sequences for changes that cannot be done in one statement on a live database. The core principles and the destructive-op gate live in `SKILL.md`; this file holds the mechanics.

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
