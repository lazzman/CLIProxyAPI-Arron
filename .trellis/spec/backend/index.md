# Backend Development Guidelines

> Repository-specific backend conventions for `github.com/router-for-me/CLIProxyAPI/v6`.

---

## Overview

This repository is a single Go module built around:

- `gin` for HTTP routing
- `logrus` for process and request logging
- `database/sql` plus `pgx` for the optional PostgreSQL-backed store
- `sdk/` packages for reusable runtime and API surfaces
- `internal/` packages for application-private implementation

These pages document what the repository does today, including current trade-offs,
so Trellis agents can match the existing codebase instead of inventing a cleaner
but incompatible architecture.

---

## Guidelines Index

| Guide | Description | Status |
|-------|-------------|--------|
| [Directory Structure](./directory-structure.md) | Module organization and file layout | Completed |
| [Database Guidelines](./database-guidelines.md) | Persistence backends, SQL patterns, bootstrap rules | Completed |
| [Error Handling](./error-handling.md) | Error wrapping, HTTP response shaping, recovery | Completed |
| [Quality Guidelines](./quality-guidelines.md) | Review standards, tests, restricted paths, verification | Completed |
| [Logging Guidelines](./logging-guidelines.md) | Shared logger setup, request logs, sensitive-data rules | Completed |

---

## Pre-Development Checklist

Read these pages before coding, based on the area you are changing:

- Always read [Directory Structure](./directory-structure.md) before adding a
  new package, file, handler, or entrypoint.
- Read [Database Guidelines](./database-guidelines.md) when touching
  `internal/store/`, persistence bootstrap, config/auth mirroring, or SQL.
- Read [Error Handling](./error-handling.md) when changing HTTP responses,
  panic recovery, or context-rich Go error returns.
- Read [Logging Guidelines](./logging-guidelines.md) when changing request
  logging, process logging, request IDs, or log destinations.
- Read [Quality Guidelines](./quality-guidelines.md) before final verification
  so build, test, and repository guardrails match current project reality.

If a task spans multiple areas, read all matching guides before implementation.

---

## Scope Notes

- Prefer these backend specs over generic Go advice when working in this repo.
- Treat `internal/` vs `sdk/` as a real boundary, not a naming preference.
- The repository mixes runtime code, CLI entrypoints, optional storage backends,
  and embeddable SDK pieces; new work should preserve those boundaries.

## Primary Code Anchors

- Entrypoint and runtime wiring: `cmd/server/main.go`
- Shared API server construction: `internal/api/server.go`
- Private app internals: `internal/*`
- Reusable API and runtime surface: `sdk/api/`, `sdk/cliproxy/`, `sdk/logging/`
- Optional persistence backends: `internal/store/postgresstore.go`, `internal/store/objectstore.go`
- Request and process logging: `internal/logging/`
