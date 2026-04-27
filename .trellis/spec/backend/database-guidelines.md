# Database Guidelines

> Persistence conventions for the optional storage backends used by this repository.

---

## Overview

This repository is not ORM-driven. The default workflow is still file-based
config and auth storage, and optional persistent backends mirror that file
layout instead of replacing it with a separate domain model.

Current persistence backends:

- Local files under `config/` and `auths/`
- PostgreSQL via `database/sql` and the `pgx` stdlib driver
  (`internal/store/postgresstore.go`)
- S3-compatible object storage via MinIO client
  (`internal/store/objectstore.go`)
- Git-backed persistence for some auth/config workflows
  (`internal/store/gitstore.go`)

If a task says "database" in this repo, it usually means the PostgreSQL-backed
store in `internal/store/postgresstore.go`, not a global ORM layer.

---

## Query Patterns

### Use `database/sql` directly

- PostgreSQL access is implemented with `database/sql` plus
  `_ "github.com/jackc/pgx/v5/stdlib"`.
- Keep SQL close to the store method that owns it.
- Always pass `context.Context` into database calls.

Real examples:

- `NewPostgresStore` validates DSN and pings the database before returning:
  `internal/store/postgresstore.go`
- `EnsureSchema` creates the schema and tables in code:
  `internal/store/postgresstore.go`
- `persistAuth` and `persistConfig` use `ExecContext` with placeholders and
  `ON CONFLICT` upserts:
  `internal/store/postgresstore.go`

### Parameterize values; only interpolate trusted identifiers

- Runtime values use placeholders such as `$1`, `$2`.
- The only interpolated SQL fragments are schema/table identifiers, and those
  must go through `quoteIdentifier` and `fullTableName`.

Real examples:

- `SELECT content FROM %s WHERE id = $1`
- `INSERT INTO %s (id, content, created_at, updated_at) ... ON CONFLICT (id)`

Both live in `internal/store/postgresstore.go`.

### Preserve the local mirror

- Persistent backends do not bypass the file layout used by the rest of the app.
- PostgreSQL mirrors data into a local spool directory containing:
  - `config/config.yaml`
  - `auths/...`
- Object storage follows the same pattern.

Real examples:

- PostgreSQL spool initialization: `internal/store/postgresstore.go`
- Object-store spool initialization: `internal/store/objectstore.go`

---

## Bootstrap And Schema Management

There is no standalone migrations directory or migration runner in this repo.
Schema creation happens inside application code during bootstrap.

Current flow:

1. `cmd/server/main.go` detects storage backend configuration from environment.
2. `store.NewPostgresStore(...)` creates the local spool directories and opens
   the DB connection.
3. `pgStoreInst.Bootstrap(...)` runs `EnsureSchema`, seeds config from
   `config.example.yaml` when needed, and mirrors auth/config into the spool.

Implications:

- Do not add a migration framework unless the task explicitly changes the
  repository's persistence model.
- Small schema changes should usually be expressed in `EnsureSchema` and the
  related bootstrap/read-write methods.
- Startup must remain able to recover from an empty database by seeding from
  the local example config.

---

## Naming Conventions

Current PostgreSQL defaults:

- Schema: optional, provided through `PostgresStoreConfig.Schema`
- Config table: `config_store`
- Auth table: `auth_store`
- Config primary key: `id = "config"`
- Content columns:
  - config content is stored as `TEXT`
  - auth content is stored as `JSONB`
- Timestamp columns:
  - `created_at`
  - `updated_at`

Path and identifier conventions:

- Auth file IDs are normalized to slash-separated relative paths.
- Auth paths are validated to stay under the managed `auths/` directory.
- Spool directories are absolute paths resolved up front.

Real examples:

- `defaultConfigTable`, `defaultAuthTable`, `defaultConfigKey`
- `relativeAuthID`, `absoluteAuthPath`, `normalizeAuthID`

All are in `internal/store/postgresstore.go`.

---

## Error And Data Handling Rules

- Prefix errors with the subsystem name, for example `postgres store: ...` or
  `object store: ...`.
- Normalize config content before persistence when the current code already does
  so. PostgreSQL config writes call `normalizeLineEndings(...)`.
- Treat missing files as delete operations where the current store logic does so
  (`PersistConfig`, `syncAuthFile`).
- Keep path-safety checks in place when translating between auth IDs and files.

---

## Common Mistakes

- Do not introduce an ORM for one feature. The current repository uses direct
  SQL and mirrored files, and new ORM-only patterns would not fit.
- Do not build SQL by concatenating runtime values. Use placeholders for values
  and quote helper functions for identifiers.
- Do not write directly to PostgreSQL or object storage while skipping the local
  spool files; other parts of the application still rely on that workspace.
- Do not remove path normalization or directory-boundary checks when adding new
  auth persistence behavior.
