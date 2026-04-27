# Directory Structure

> How backend code is organized in this repository today.

---

## Overview

This is a single-module Go backend. The highest-level split is:

- `cmd/` for binaries and startup wiring
- `internal/` for application-private implementation
- `sdk/` for reusable public runtime, API, and helper packages
- `test/` for cross-package or black-box tests
- `docs/` and `examples/` for consumer-facing material, not server internals

The most important architectural rule is the `internal/` vs `sdk/` boundary:
application-only code stays under `internal/`, while reusable surfaces that
examples or external embedders may import live under `sdk/`.

---

## Directory Layout

```text
cmd/
  server/
  fetch_antigravity_models/
  validate_models_catalog/
internal/
  access/
  api/
  auth/
  browser/
  buildinfo/
  cache/
  cmd/
  config/
  constant/
  interfaces/
  logging/
  managementasset/
  misc/
  registry/
  runtime/
  store/
  thinking/
  tui/
  usage/
  util/
  watcher/
  wsrelay/
sdk/
  access/
  api/
  auth/
  cliproxy/
  config/
  logging/
  proxyutil/
  translator/
test/
docs/
examples/
```

---

## Placement Rules

### `cmd/` owns executable entrypoints

- Put binary-specific startup code in `cmd/<tool>/main.go`.
- `cmd/server/main.go` is the composition root for the server process: flag
  parsing, environment loading, storage backend selection, config bootstrap,
  and startup sequencing all happen there.
- Keep `cmd/` focused on wiring. Reusable behavior should move into `internal/`
  or `sdk/`, not grow inside `main.go`.

### `internal/` owns private application implementation

- `internal/api/` contains the Gin server, middleware, route registration, and
  management handlers.
  Example: `internal/api/server.go`, `internal/api/handlers/management/handler.go`
- `internal/api/modules/` is the extension point for optional route bundles.
  Example: `internal/api/modules/modules.go`, `internal/api/modules/amp/routes.go`
- `internal/store/` contains persistence backends and their local spool logic.
  Example: `internal/store/postgresstore.go`, `internal/store/objectstore.go`
- `internal/logging/` owns shared process logging, Gin middleware, request IDs,
  request-log file writing, and log retention.
- `internal/registry/` and `internal/thinking/` hold project-private model
  metadata and provider-thinking logic that support runtime behavior but are
  not exposed as stable SDK surface.
- `internal/access/`, `internal/config/`, and `internal/util/` contain shared
  private helpers for filesystem access, config normalization, and reusable
  low-level utilities used across the server/runtime code.
- `internal/runtime/`, `internal/auth/`, and `internal/watcher/` hold provider
  execution, auth flows, and hot-reload/watch internals that are not exposed as
  stable external packages.
- `internal/browser/`, `internal/tui/`, and `internal/managementasset/` support
  local management and embedded assets for the server process, so they still
  belong under `internal/` rather than a public SDK layer.

### `sdk/` owns reusable API/runtime packages

- `sdk/api/handlers/` holds provider-facing API handlers and shared handler
  primitives used by the server and embedders.
  Example: `sdk/api/handlers/handlers.go`, `sdk/api/handlers/openai/openai_handlers.go`
- `sdk/cliproxy/` contains the embeddable service builder/runtime surface.
  Example: `sdk/cliproxy/builder.go`
- `sdk/logging/` and `sdk/auth/` are thin public-facing wrappers around
  reusable pieces implemented internally.
- `sdk/access/`, `sdk/config/`, and `sdk/translator/` expose reusable helpers
  that external consumers can import without depending on `internal/`.

### Tests stay close to the package, with extra integration coverage in `test/`

- Most tests are colocated as `*_test.go` next to the code they verify.
  Example: `internal/logging/gin_logger_test.go`
  Example: `sdk/api/handlers/openai/openai_responses_handlers_stream_test.go`
- The top-level `test/` directory is used for broader cross-package scenarios.
  Example: `test/amp_management_test.go`

---

## Naming Conventions

- Package directories are lowercase and usually short: `internal/store`,
  `sdk/cliproxy`, `internal/watcher/diff`.
- Go source files are lowercase with underscores when needed:
  `postgresstore.go`, `request_logging.go`, `openai_responses_websocket.go`.
- Tests use the standard Go `*_test.go` suffix.
- Entrypoints use `main.go` under a tool-specific directory:
  `cmd/server/main.go`, `cmd/validate_models_catalog/main.go`.
- Provider-specific HTTP handlers are grouped by provider under
  `sdk/api/handlers/<provider>/`.

---

## Real Examples

- Composition root:
  `cmd/server/main.go`
- Private shared config/helper layers:
  `internal/config/config.go`, `internal/access/config_access/provider.go`
- Private server assembly:
  `internal/api/server.go`
- Optional route module registration:
  `internal/api/modules/modules.go`
- Private model/runtime support packages:
  `internal/registry/model_registry.go`, `internal/thinking/provider/openai/apply.go`
- Provider-specific handler package:
  `sdk/api/handlers/openai/openai_handlers.go`
- Reusable service builder:
  `sdk/cliproxy/builder.go`
- Reusable public config/access surface:
  `sdk/config/config.go`, `sdk/access/types.go`
- Private persistence backend:
  `internal/store/postgresstore.go`

---

## Anti-Patterns To Avoid

- Do not put new reusable SDK surface under `internal/` if examples or external
  consumers will need to import it.
- Do not grow `cmd/server/main.go` with business logic that can live in a
  package-level type or helper.
- Do not place management-only handlers under `sdk/api/handlers/`; they belong
  under `internal/api/handlers/management/`.
- Do not invent a new top-level `pkg/` or `src/` layer. The repository already
  standardizes on `cmd/`, `internal/`, `sdk/`, and `test/`.
