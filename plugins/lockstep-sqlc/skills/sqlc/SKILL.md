---
name: sqlc
description: Enforce sqlc codegen conventions for Go + PostgreSQL. Use when editing sqlc.yaml, **/queries/*.sql, or generated *.sql.go, or when the user mentions sqlc, codegen, Querier interface, db.Queries, sqlc generate, or sqlc vet. Keeps SQL as the source of truth, routes every call through the generated Querier, uses v2 config, and regenerates rather than hand-editing generated files.
paths:
  - "**/sqlc.yaml"
  - "**/sqlc.yml"
  - "**/queries/*.sql"
  - "**/queries/**/*.sql"
allowed-tools:
  - Read
  - Grep
---

> Targets sqlc 1.31+ · verified 2026-07 (latest v1.31.1, released 2026-04-22).

This skill enforces sqlc 1.31+ conventions for Go projects with PostgreSQL. The rule: SQL is the source of truth, Go calls go through the generated `Querier` interface only.

Apply only when sqlc is configured (`sqlc.yaml` exists). If the project uses `sqlx` / raw `database/sql` everywhere, **STOP** and ask the user before introducing sqlc.

## sqlc.yaml requirements

```yaml
version: "2"
sql:
  - engine: postgresql
    queries: db/queries
    schema: db/migrations
    gen:
      go:
        package: db
        out: db/sqlc
        emit_interface: true
        emit_json_tags: true
        emit_db_tags: false
        emit_pointers_for_null_types: true
        emit_empty_slices: true
        sql_package: pgx/v5
```

- **`version: "2"`** required. v1 is deprecated.
- **`emit_interface: true`** generates the `Querier` interface so handlers can mock easily.
- **`sql_package: pgx/v5`** preferred over `database/sql` for PostgreSQL.
- **`schema:` points to migrations**, not to a hand-written schema file. sqlc parses migrations to derive the schema.

## Query naming convention

| Prefix | Returns | Example |
|---|---|---|
| `Get` | exactly one row, error if missing | `GetUserByID` |
| `Find` | one row or `nil`, no error if missing | `FindUserByEmail` |
| `List` | many rows | `ListUsersByOrg` |
| `Count` | scalar count | `CountActiveUsers` |
| `Create` | inserts and returns the created row | `CreateUser` |
| `Update` | updates and returns the updated row | `UpdateUserEmail` |
| `Delete` | deletes, no return | `DeleteUser` |

Each query starts with `-- name: <PascalCase> :one|:many|:exec|:execrows`:

```sql
-- name: GetUserByID :one
SELECT id, email, created_at FROM users WHERE id = $1;

-- name: ListUsersByOrg :many
SELECT id, email FROM users WHERE org_id = $1 ORDER BY created_at DESC;

-- name: CreateUser :one
INSERT INTO users (email, org_id) VALUES ($1, $2) RETURNING *;
```

## Forbidden patterns

- Hand-written `db.Query("SELECT ...")` / `db.Exec("INSERT ...")` from Go when sqlc could generate it. Add a query to `db/queries/*.sql` and run `sqlc generate`.
- `version: "1"` in `sqlc.yaml`. Migrate to v2.
- Modifying generated files in `db/sqlc/` (e.g. `models.go`, `queries.sql.go`). They will be overwritten. Modify the SQL source and regenerate.
- Defining the schema twice (once in migrations, once in a separate `schema.sql`). Point sqlc at migrations.
- Mixing pgx and `database/sql` in the same project. Pick one—`pgx/v5` preferred.
- String concatenation to build dynamic queries. Use `sqlc.arg()`, `sqlc.embed()`, or write multiple named queries.
- Query names that do not start with one of the prefixes above. `FetchUser`, `RetrieveUserByEmail` — pick the canonical prefix.
- A new query when an existing one in `db/queries/*.sql` already covers the use case. grep query names and the underlying `SELECT/INSERT/UPDATE/DELETE` first; reuse or extend if found.

## When sqlc cannot express what you need

sqlc handles most CRUD plus aggregates, CTEs, and window functions. If you need:
- runtime dynamic column selection
- variable-arity `IN ()` clauses without an array
- prepared statement caching control

**STOP** and report:

> Need to express [query shape]. sqlc patterns checked: [`sqlc.arg`, `sqlc.embed`, `ANY($1::int[])`]. None covers [specific gap]. Approve one of: (A) restructure with `ANY(?::type[])` for `IN`, (B) add a hand-written method on the `*db.Queries` receiver in a separate non-generated file (with comment), (C) different approach.

## After every SQL change

1. Run `sqlc vet` (linting against the schema)
2. Run `sqlc generate`
3. Commit both the SQL change and the generated Go output
4. Run `go test ./...`

## Verification (grep after every sqlc-related change)

```bash
# config version
grep -nE '^version:\s*"?1' **/sqlc.yaml **/sqlc.yml 2>/dev/null

# hand-written SQL calls (should be sqlc queries)
grep -rnE 'db\.(Query|QueryRow|Exec)\(["`]\s*(SELECT|INSERT|UPDATE|DELETE)' --include='*.go' .

# query naming convention
grep -nE '^-- name:\s+\w+' db/queries/*.sql | grep -vE ':\s+(Get|Find|List|Count|Create|Update|Delete)\w+'
```

Reference: https://docs.sqlc.dev/en/latest/reference/config.html
